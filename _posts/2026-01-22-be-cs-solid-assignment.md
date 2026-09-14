---
title: "SOLID 과제"
date: 2026-01-22 00:07:00 +0900
categories: [cs, oop]
tags: [oop, solid, java]
---

**기존 코드**

```java
import org.kohsuke.github.*;
import java.io.IOException;
public class RepositoryRank {
  public int getPoint(String repositoryName) throws IOException {
    GitHub github = GitHub.connect();
    GHRepository repository = github.getRepository(repositoryName);
    int points = 0;
    if (repository.hasIssues() ) {
      points += 1;
    }
    if (repository.getReadme() != null) {
      points += 1;
    }
    if (repository.getPullRequests(GHIssueState.CLOSED).size() > 0) {
      points += 1;
    }
    points += repository.getStargazersCount();
    points += repository.getForksCount();
    return points;
  }
  public static void main(String[] args) throws IOException {
    RepositoryRank spring = new RepositoryRank();
    int point = spring.getPoint("whiteship/live-study");
    System.out.println(point);
  }
}
```

- S
- O - **TODO**
- L
- I
- D

```java
package com.hoho.domain.auth.service;

import lombok.RequiredArgsConstructor;
import org.kohsuke.github.*;
import java.io.IOException;

@RequiredArgsConstructor
public class RepositoryRank {
    private final Github github; // DI

    // RequiredArgsConstructor 어노테이션 안쓰면 붙이기
//    public RepositoryRank(Github github) {
//        this.github = github;
//    }

    public int getPoint(String repositoryName) throws IOException {

        GHRepository repository = github.getRepository(repositoryName);
        int points = calculatePoints(repository);
        return points;
    }

    // SRP
    private static int calculatePoints(GHRepository repository) {
        int points = 0;
        if (repository.hasIssues() ) {
            points += 1;
        }
        if (repository.getReadme() != null) {
            points += 1;
        }
        if (repository.getPullRequests(GHIssueState.CLOSED).size() > 0) {
            points += 1;
        }
        points += repository.getStargazersCount();
        points += repository.getForksCount();
        return points;
    }

    public interface PointRule {
        int getPoint(GHRepository repository) throws IOException;
    }

    // 이슈 있으면 1점
    public class IssueRule implements PointRule {
        @Override
        public int getPoint(GHRepository repository) {
            return repository.hasIssues() ? 1 : 0;
        }
    }

    // 스타 개수만큼 점수
    public class StarRule implements PointRule {
        @Override
        public int getPoint(GHRepository repository) {
            return repository.getStargazersCount();
        }
    }

    // Fork 개수만큼 점수
    public class ForkRule implements PointRule {
        @Override
        public int getPoint(GHRepository repository) {
            return repository.getForksCount();
        }
    }

    public static void main(String[] args) throws IOException {
        RepositoryRank spring = new RepositoryRank();
        int point = spring.getPoint("whiteship/live-study");
        System.out.println(point);
    }
}

```
