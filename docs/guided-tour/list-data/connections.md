---
id: connections
title: Connections
slug: /guided-tour/list-data/connections/
description: Relay guide for connections
keywords:
- pagination
- connections
---

import DocsRating from '@site/src/core/DocsRating';
import {OssOnly, FbInternalOnly} from 'internaldocs-fb-helpers';
import useBaseUrl from '@docusaurus/useBaseUrl';

GraphQL 서버로부터 데이터의 리스트를 쿼리하고 싶은 여러 시나리오가 있을 것입니다. 보통 모든 데이터를 미리 쿼리하고 싶지는 않을 것이고, 오히려 유저 인풋이나 다른 이벤트의 응답으로써 그 리스트의 일부분만을 점진적으로 쿼리하고 싶을 것입니다. 이렇게 데이터의 리스트를 부분으로 나눠서 쿼리하는 것을 일반적으로 [페이지네이션](https://graphql.org/learn/pagination/)이라고 합니다.

구체적으로, 릴레이에서는 [커넥션](https://graphql.org/learn/pagination/#complete-connection-model)이라는 필드를 통해 페이지네이션을 합니다. 커넥션은 GraphQL 필드이고, 리스트에서 어떤 슬라이스를 쿼리하고 싶은지를 인자로 받습니다. 그 응답에는 요청된 리스트의 슬라이스와, 그 이후에 데이터가 더 있는지, 더 있다면 어떻게 하면 가져올 수 있는지에 대한 정보가 담깁니다. 이 추가 정보를 이용하여 그 다음 슬라이스(혹은 페이지)를 쿼리하면 페이지네이션을 수행할 수 있습니다.

더 자세하게는 *커서 기반 페이지네이션*이 사용됩니다. 커서 기반 페이지네이션이란 리스트의 슬라이스를 쿼리하는 데에 커서(`cursor`)와 개수(`count`)를 사용하는 것을 말합니다. 커서는 근본적으로 리스트의 특정 위치를 가리키는 마커(혹은 포인터)의 역할을 하는 불투명한(opaque) 토큰입니다. 커서 기반 페이지네이션과 커넥션에 대해 더 자세히 알고 싶다면 <a href={useBaseUrl('graphql/connections.htm')}>명세</a>를 참고하세요.


<DocsRating />
