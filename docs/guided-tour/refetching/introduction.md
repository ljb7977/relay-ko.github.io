---
id: introduction
title: Introduction
slug: /guided-tour/refetching/
description: Relay guide to refetching
keywords:
- refetching
---

import DocsRating from '@site/src/core/DocsRating';
import {OssOnly, FbInternalOnly} from 'internaldocs-fb-helpers';

앱이 처음에 렌더링되고 나면, 유저 인터랙션이나 이벤트의 결과로서 데이터를 다시 가져와서 (refetch) 새롭고 다른 데이터를 보여주거나 (예: 현재 표시하는 아이템을 바꾸기), 현재 렌더링되는 데이터를 서버의 최신 버전으로 새로고침하는 (예: 개수 새로고침) 여러 시나리오가 보통 있습니다.

이번 섹션에서는 가장 자주 일어나는 시나리오와, 그들을 릴레이로 구현하는 방법을 다룰 것입니다.

<DocsRating />
