---
title: "역할 기반 접근 제어(RBAC)"
date: 2026-02-15 17:43:00 +0900
categories: [spring, security]
tags: [spring-security, rbac, jpa]
---

### JPA기반 RBAC(Role-Based Access Control)

RBAC는 사용자에게 직접 권한을 부여하는 대신 Role을 매개로 권한을 관리하는 접근 제어 모델이다. Spring Security + JPA를 결함하여 구현한다.

### 1. RBAC의 개념

RBAC는 세 가지 핵심 엔티티의 관계로 구성된다.

```java
User <- N:M -> Role <N:M> Permisssion
```

- **User**: 시스템 사용자
- **Role**: 역할 (ADMIN, MANAGER, USER 등)
- **Permission**: 세부 권한 (READ_POST, WRITE_POST, DELETE_USER 등)

사용자는 여러 역할을 가질 수 있고, 각 역할은 여러 권한을 포함한다. 이것이 RBAC Level 1 (Flat RBAC)이며, 여기에 역할 간 계층 구조를 추가하면 RBAC Level 2 (Hierarchical RBAC)가 된다.

### JPA 엔티티 설계 예시

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false)
    private String password;

    private boolean enabled = true;

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();

    // getters, setters
}
```

```java
@Entity
@Table(name = "roles")
public class Role {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String name;  // ROLE_ADMIN, ROLE_USER 등

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "role_permissions",
        joinColumns = @JoinColumn(name = "role_id"),
        inverseJoinColumns = @JoinColumn(name = "permission_id")
    )
    private Set<Permission> permissions = new HashSet<>();

    // getters, setters
}
```

```java
@Entity
@Table(name = "permissions")
public class Permission {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String name;  // READ_POST, WRITE_POST, DELETE_USER 등

    // getters, setters
}
```

### Spring Security 연동

**코드 예시**

Spring Security가 인증 시 사용자 정보를 로드하는 핵심 부분이다. 여기서 Role과 Permission을 모두 GrantedAuthority로 변환한다.

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        return new org.springframework.security.core.userdetails.User(
            user.getUsername(),
            user.getPassword(),
            user.isEnabled(),
            true, true, true,
            getAuthorities(user.getRoles())
        );
    }

    private Collection<? extends GrantedAuthority> getAuthorities(Set<Role> roles) {
        List<GrantedAuthority> authorities = new ArrayList<>();

        for (Role role : roles) {
            // 역할 자체를 authority로 추가 (ROLE_ 접두사)
            authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getName()));

            // 각 역할의 세부 권한도 authority로 추가
            for (Permission permission : role.getPermissions()) {
                authorities.add(new SimpleGrantedAuthority(permission.getName()));
            }
        }

        return authorities;
    }
}
```

### 권한 검사 방법

- 방법 A: URL 기반 (SecurityFilterChain)

  위의 `SecurityConfig`처럼 URL 패턴에 역할/권한을 매핑한다.

- 방법 B: 메서드 수준 어노테이션

  ```java
  @RestController
  @RequestMapping("/api/posts")
  public class PostController {

      // 역할 기반 검사
      @GetMapping
      @PreAuthorize("hasRole('USER')")
      public List<Post> getAllPosts() { ... }

      // 권한 기반 검사
      @PostMapping
      @PreAuthorize("hasAuthority('WRITE_POST')")
      public Post createPost(@RequestBody Post post) { ... }

      // 복합 조건
      @DeleteMapping("/{id}")
      @PreAuthorize("hasRole('ADMIN') or (hasAuthority('DELETE_POST') and #post.author == authentication.name)")
      public void deletePost(@PathVariable Long id) { ... }

      // 여러 권한 중 하나 만족
      @PutMapping("/{id}")
      @PreAuthorize("hasAnyAuthority('WRITE_POST', 'EDIT_POST')")
      public Post updatePost(@PathVariable Long id, @RequestBody Post post) { ... }
  }
  ```
