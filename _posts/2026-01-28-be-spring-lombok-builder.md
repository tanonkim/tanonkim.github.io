---
title: "Lombok.builder"
date: 2026-01-28 21:34:00 +0900
categories: [spring, core]
tags: [lombok, builder]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

```java
package com.example.member.controller.request;

import com.example.model.enum_type.PolicySort;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.ToString;
import org.springframework.data.domain.PageRequest;

@Getter
@Builder
@NoArgsConstructor
@ToString
// @AllArgsConstructor
public class MemberPolicySearch {

  private Long memberId;

	// @Default 제거함
  private String categoryCode;

	// @Default 제거함
  private PolicySort policySort = PolicySort.POPULARITY;

	// @Default 제거함
  private Boolean activeYN = false;

	// @Default 제거함
  private Boolean newYN = false;

	// @Default 제거함
  private Boolean closingSoonYN = false;

	// @Default 제거함
  private Boolean popularYN = false;

	// @Default 제거함
  private Boolean unreadYN = false;

	// @Default 제거함
  private int page = 1;

	// @Default 제거함
  private int size = 50;

  // todo : 확인
  public MemberPolicySearch(Long memberId, String categoryCode, PolicySort policySort,
      Boolean activeYN, Boolean newYN, Boolean closingSoonYN, Boolean popularYN,
      Boolean unreadYN, int page, int size) {
    this.memberId = memberId;
    this.categoryCode = categoryCode;
    this.policySort = (policySort != null) ? policySort : PolicySort.POPULARITY;
    this.activeYN = (activeYN != null) ? activeYN : false;
    this.newYN = (newYN != null) ? newYN : false;
    this.closingSoonYN = (closingSoonYN != null) ? closingSoonYN : false;
    this.popularYN = (popularYN != null) ? popularYN : false;
    this.unreadYN = (unreadYN != null) ? unreadYN : false;
    this.page = page;
    this.size = size;
  }

  public PageRequest pageRequest() {
    return PageRequest.of(page - 1, size);
  }

}

```
