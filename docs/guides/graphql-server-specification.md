---
id: graphql-server-specification
title: GraphQL Server Specification
slug: /guides/graphql-server-specification/
description: Relay GraphQL server specification guide
keywords:
- GraphQL
- server
- specification
---

import DocsRating from '@site/src/core/DocsRating';

이 문서의 목적은 릴레이가 GraphQL 서버에 대해 하는 가정을 설명하고, GraphQL 예제 스키마를 통해 그를 설명하기 위함입니다.

목차:

-   [서론](#preface)
-   [스키마](#schema)
-   [객체 식별](#object-identification)
-   [커넥션](#connections)
-   [더 읽어보기](#further-reading)

## 서론

릴레이는 GraphQL 서버가 다음 두 개를 제공한다고 가정합니다.

1.  객체를 리페치하기 위한 구조
2.  커넥션을 페이지네이션하는 방법에 대한 설명 

이 예제는 이 두 가정을 모두 설명합니다. 이 예제는 포괄적이진 않지만, 라이브러리의 자세한 스펙에 대해 파고 들어가기 전에 맥락을 조금 설명하기 위해, 이러한 핵심 가정을 빠르게 소개하도록 설계되었습니다.

이 예제에서는 GraphQL을 사용하여 오리지널 스타워즈 3부작에 나오는 함선(ship)과 세력(faction)에 대한 정보를 쿼리합니다.

이 글을 읽는 여러분들이 이미 [GraphQL](http://graphql.org/)에 익숙하다고 가정합니다. 그렇지 않다면, [GraphQL.js](https://github.com/graphql/graphql-js)의 README부터 시작하면 좋습니다.

또한 여러분들이 [스타워즈](https://en.wikipedia.org/wiki/Star_Wars)에 익숙하다고 가정할 것입니다. 그렇지 않다면, 1977년 버전의 스타워즈를 보면 좋습니다. (1997년의 스타워즈 스페셜 에디션이 이 문서의 목적에 알맞긴 하지만요)

## 스키마

아래의 스키마는 릴레이에서 사용하는 GraphQL 서버가 구현해야 하는 기능을 설명하는 데에 사용됩니다. 두 가지 핵심 타입은 스타워즈 유니버스의 세력과 함선으로, 각 세력은 많은 함선을 가지고 있습니다.


```graphql
interface Node {
  id: ID!
}

type Faction implements Node {
  id: ID!
  name: String
  ships: ShipConnection
}

type Ship implements Node {
  id: ID!
  name: String
}

type ShipConnection {
  edges: [ShipEdge]
  pageInfo: PageInfo!
}

type ShipEdge {
  cursor: String!
  node: Ship
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type Query {
  rebels: Faction
  empire: Faction
  node(id: ID!): Node
}
```

## 객체 식별

`Faction`과 `Ship`은 그들을 리페치하는 데에 사용할 수 있는 식별자를 가집니다. 우리는 이 능력을 `Node` 인터페이스와 루트 쿼리 타입의 `node` 필드를 통해 릴레이에게 노출시킵니다. 

`Node` 인터페이스는 `id`라는 필드 하나를 포함하고, 그 타입은 `ID!`입니다. `node` 루트 필드는 `ID!` 타입의 인자 하나를 받으며, `Node`를 반환합니다. 이 둘은 리페치를 가능하게 하기 위해 함께 작동합니다. 이전에 Node에서 받았던 `id`를 `node`에 넘기면, 그 객체를 다시 받습니다.

이것을 실제로 해 봅시다. 반란 연합(rebels)의 ID를 쿼리해 봅시다.

```graphql
query RebelsQuery {
  rebels {
    id
    name
  }
}
```

이 쿼리는 아래 결과를 반환합니다.

```json
{
  "rebels": {
    "id": "RmFjdGlvbjox",
    "name": "Alliance to Restore the Republic"
  }
}
```

반란 연합의 ID를 알았으니, 이제 이를 다시 가져올 수 있습니다.

```graphql
query RebelsRefetchQuery {
  node(id: "RmFjdGlvbjox") {
    id
    ... on Faction {
      name
    }
  }
}
```

이는 다음을 반환합니다.

```json
{
  "node": {
    "id": "RmFjdGlvbjox",
    "name": "Alliance to Restore the Republic"
  }
}
```

은하 제국(empire)에 대해서도 똑같이 하면, 다른 ID가 반환되는 것을 알 수 있습니다. 이 또한 다시 가져오기를 할 수 있습니다.

```graphql
query EmpireQuery {
  empire {
    id
    name
  }
}
```

는 다음을 반환하고,

```json
{
  "empire": {
    "id": "RmFjdGlvbjoy",
    "name": "Galactic Empire"
  }
}
```

아래 쿼리는

```graphql
query EmpireRefetchQuery {
  node(id: "RmFjdGlvbjoy") {
    id
    ... on Faction {
      name
    }
  }
}
```

이런 결과를 반환합니다.

```json
{
  "node": {
    "id": "RmFjdGlvbjoy",
    "name": "Galactic Empire"
  }
}
```

`Node` 인터페이스와 `node` 필드는 이런 다시 가져오기에 사용하는 ID가 전역적으로 유일하다고 가정합니다. 전역적으로 유일한 ID가 없는 시스템은 일반적으로 타입과 그 타입에서의 ID를 합쳐서 전역적으로 유일한 ID를 만들 수 있습니다. (이 예제에서도 그런 방법이 사용되었습니다.)

여기서 돌려받은 ID들은 base64 문자열입니다. ID는 불투명(즉, `node` 쿼리에 넣는 인자 `id`는 시스템의 어떤 객체를 조회해서 얻은 `id`를 바꾸지 않고 그대로 다시 사용해야 합니다)하도록 설계되었습니다. 문자열을 base64로 인코딩하면 보는 사람이 이 문자열이 불투명한 식별자라는 것을 알 수 있기 때문에, 이것은 GraphQL에서 사용하는 유용한 컨벤션입니다.

서버가 어떻게 동작해야 하는지에 대한 모든 세부 사항은 [GraphQL 객체 식별](https://graphql.org/learn/global-object-identification/) 모범 사례 가이드에서 확인할 수 있습니다.

## 커넥션

스타워즈 유니버스에서 세력은 여러 함선을 가지고 있습니다. 릴레이에는 일대다 관계를 표현하는 표준화된 방법을 통하여 그러한 일대다 관계를 쉽게 조작하도록 만드는 기능이 있습니다. 이 표준 커넥션 모델은 커넥션을 페이지네이션하고, 자르는 방법을 제공합니다. 

반란 연합을 예로 들어서, 그들의 첫 번째 함선이 뭔지 물어봅시다.

```graphql
query RebelsShipsQuery {
  rebels {
    name
    ships(first: 1) {
      edges {
        node {
          name
        }
      }
    }
  }
}
```

결과:

```json
{
  "rebels": {
    "name": "Alliance to Restore the Republic",
    "ships": {
      "edges": [
        {
          "node": {
            "name": "X-Wing"
          }
        }
      ]
    }
  }
}
```

결과에서 첫 번째 하나만 자르기 위해 `ships`의 `first` 인자를 사용했습니다. 여기서 페이지네이션을 하고 싶다면 어떻게 할까요? 각 엣지에는 페이지네이션에 사용할 수 있는 커서가 제공됩니다. 이번엔 첫 두 개를 요청하면서 커서도 받아봅시다.

```
query MoreRebelShipsQuery {
  rebels {
    name
    ships(first: 2) {
      edges {
        cursor
        node {
          name
        }
      }
    }
  }
}
```

결과는 다음과 같습니다.

```json

{
  "rebels": {
    "name": "Alliance to Restore the Republic",
    "ships": {
      "edges": [
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjA=",
          "node": {
            "name": "X-Wing"
          }
        },
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjE=",
          "node": {
            "name": "Y-Wing"
          }
        }
      ]
    }
  }
}
```

커서가 base64 문자열이라는 것에 주목하세요. 이는 앞에서 봤던 패턴과 같습니다. 서버는 우리에게 이것이 불투명한 문자열이라는 것을 알려주고 있습니다. 우리는 이 문자열을 `ships` 필드의 `after` 인자에 넣어서 서버에 다시 넘길 수 있습니다. 이렇게 하면 이전 결과의 마지막 함선 다음의 함선 3개를 요청할 수 있습니다.

```

query EndOfRebelShipsQuery {
  rebels {
    name
    ships(first: 3 after: "YXJyYXljb25uZWN0aW9uOjE=") {
      edges {
        cursor
        node {
          name
        }
      }
    }
  }
}
```

결과는 다음과 같습니다.

```json


{
  "rebels": {
    "name": "Alliance to Restore the Republic",
    "ships": {
      "edges": [
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjI=",
          "node": {
            "name": "A-Wing"
          }
        },
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjM=",
          "node": {
            "name": "Millenium Falcon"
          }
        },
        {
          "cursor": "YXJyYXljb25uZWN0aW9uOjQ=",
          "node": {
            "name": "Home One"
          }
        }
      ]
    }
  }
}
```

좋네요! 계속해서 다음 4개를 받아 봅시다.

```graphql
query RebelsQuery {
  rebels {
    name
    ships(first: 4 after: "YXJyYXljb25uZWN0aW9uOjQ=") {
      edges {
        cursor
        node {
          name
        }
      }
    }
  }
}
```


```json
{
  "rebels": {
    "name": "Alliance to Restore the Republic",
    "ships": {
      "edges": []
    }
  }
}
```

흠. 더 이상 없네요. 반란 연합의 함선은 5개 밖에 없었던 것 같습니다. 이렇게 서버에 한번 더 다녀와서 확인하지 않고도 커넥션의 마지막에 다다랐다는 것을 미리 알았다면 좋았을 것입니다. 커넥션 모델은 `PageInfo`라는 타입을 통해 이를 지원합니다. 함선을 조회하기 위해 썼던 두 쿼리를 다시 써 봅시다. 이번에는 `hasNextPage`도 요청하고요.

```graphql
query EndOfRebelShipsQuery {
  rebels {
    name
    originalShips: ships(first: 2) {
      edges {
        node {
          name
        }
      }
      pageInfo {
        hasNextPage
      }
    }
    moreShips: ships(first: 3 after: "YXJyYXljb25uZWN0aW9uOjE=") {
      edges {
        node {
          name
        }
      }
      pageInfo {
        hasNextPage
      }
    }
  }
}
```

결과는 다음과 같습니다.

```json
{
  "rebels": {
    "name": "Alliance to Restore the Republic",
    "originalShips": {
      "edges": [
        {
          "node": {
            "name": "X-Wing"
          }
        },
        {
          "node": {
            "name": "Y-Wing"
          }
        }
      ],
      "pageInfo": {
        "hasNextPage": true
      }
    },
    "moreShips": {
      "edges": [
        {
          "node": {
            "name": "A-Wing"
          }
        },
        {
          "node": {
            "name": "Millenium Falcon"
          }
        },
        {
          "node": {
            "name": "Home One"
          }
        }
      ],
      "pageInfo": {
        "hasNextPage": false
      }
    }
  }
}
```

첫 번째 쿼리에서는 다음 페이지가 있다고 알려주지만, 그 다음 쿼리에서는 커넥션의 마지막에 도달했다고 알려줍니다.

릴레이는 커넥션에 대한 추상화를 구축하기 위해 이 모든 기능을 사용하고, 클라이언트에서 커서를 직접 관리할 필요 없이 효율적으로 동작하는 것을 쉽게 만들어 줍니다.

서버가 동작해야 하는 방식에 대한 전체 세부 사항은 [GraphQL 커서 커넥션](https://relay.dev/graphql/connections.htm) 스펙을 참조하세요.

## 더 읽을 거리


이로써 GraphQL 서버 스펙의 개요를 마치겠습니다. 릴레이를 준수하는 GraphQL 서버의 자세한 요구 사항에 대해서는, [릴레이 커서 커넥션](https://relay.dev/graphql/connections.htm) 모델과 [GraphQL 전역 객체 식별](https://graphql.org/learn/global-object-identification/) 모델의 더 공식적인 설명이 제공됩니다.

이 스펙을 구현하는 코드를 보려면, [GraphQL.js 릴레이 라이브러리](https://github.com/graphql/graphql-relay-js) 가 노드와 커넥션을 만드는 헬퍼 함수를 제공합니다. 그 레포지터리의 [`__tests__`](https://github.com/graphql/graphql-relay-js/tree/main/src/__tests__) 폴더에는 위의 예시에 대한 구현이 통합 테스트의 형태로 들어 있습니다.

<DocsRating />
