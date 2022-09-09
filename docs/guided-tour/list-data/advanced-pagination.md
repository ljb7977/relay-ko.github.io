---
id: advanced-pagination
title: Advanced Pagination
slug: /guided-tour/list-data/advanced-pagination/
description: Relay guide for advanced pagination
keywords:
- pagination
- usePaginationFragment
- prefetching
---

import DocsRating from '@site/src/core/DocsRating';
import {OssOnly, FbInternalOnly} from 'internaldocs-fb-helpers';

이번 섹션에서는 `usePaginationFragment`로 다뤘던 기본 케이스보다 더 진화된 페이지네이션을 구현하는 방법을 다룰 것입니다.


## 여러 커넥션에 대한 페이지네이션

같은 컴포넌트의 여러 커넥션에 대해 페이지네이션을 해야 한다면, `usePaginationFragment`를 여러 번 사용하면 됩니다.

```js
import type {CombinedFriendsListComponent_user$key} from 'CombinedFriendsListComponent_user.graphql';
import type {CombinedFriendsListComponent_viewer$key} from 'CombinedFriendsListComponent_viewer.graphql';

const React = require('React');

const {graphql, usePaginationFragment} = require('react-relay');

type Props = {
  user: CombinedFriendsListComponent_user$key,
  viewer: CombinedFriendsListComponent_viewer$key,
};

function CombinedFriendsListComponent(props: Props) {

  const {data: userData, ...userPagination} = usePaginationFragment(
    graphql`
      fragment CombinedFriendsListComponent_user on User {
        name
        friends
          @connection(
            key: "CombinedFriendsListComponent_user_friends_connection"
          ) {
          edges {
            node {
              name
              age
            }
          }
        }
      }
    `,
    props.user,
  );

  const {data: viewerData, ...viewerPagination} = usePaginationFragment(
    graphql`
      fragment CombinedFriendsListComponent_user on Viewer {
        actor {
          ... on User {
            name
            friends
              @connection(
                key: "CombinedFriendsListComponent_viewer_friends_connection"
              ) {
              edges {
                node {
                  name
                  age
                }
              }
            }
          }
        }
      }
    `,
    props.viewer,
  );

  return (...);
}
```

하지만 한 컴포넌트에 하나의 커넥션만 쓰는 것을 추천합니다. 그래야 컴포넌트를 쉽게 이해할 수 있습니다.



## 양방향 페이지네이션


[Pagination](../pagination/) 섹션에서는 `usePaginationFragment`를 사용하여 한 개의 정방향 페이지네이션을 하는 방법을 다뤘습니다. 하지만 커넥션은 그의 반대 방향인 역방향 페이지네이션 또한 허용합니다. 정방향과 역방향이라는 말의 의미는 커넥션의 아이템의 정렬 방식에 종속적일 수 있습니다. 예를 들어, *정방향*은 더 최신을 의미하고, *역방향*은 덜 최신인 것을 의미할 수 있습니다.
방향의 의미에 관계 없이, 릴레이는 반대 방향으로 페이지네이션하는 데에도 같은 API를 제공합니다. 똑같이 `usePaginationFragment`를 사용하고, 단지 커넥션 인자 `after`와 `first` 대신에 `before`와 `last`가 사용됩니다.

```js
import type {FriendsListComponent_user$key} from 'FriendsListComponent_user.graphql';

const React = require('React');
const {Suspense} = require('React');

const {graphql, usePaginationFragment} = require('react-relay');

type Props = {
  userRef: FriendsListComponent_user$key,
};

function FriendsListComponent(props: Props) {
  const {
    data,
    loadPrevious,
    hasPrevious,
    // ... forward pagination values
  } = usePaginationFragment(
    graphql`
      fragment FriendsListComponent_user on User {
        name
        friends(after: $after, before: $before, first: $first, last: $last)
          @connection(key: "FriendsListComponent_user_friends_connection") {
          edges {
            node {
              name
              age
            }
          }
        }
      }
    `,
    userRef,
  );

  return (
    <>
      <h1>Friends of {data.name}:</h1>
      <List items={data.friends?.edges.map(edge => edge.node)}>
        {node => {
          return (
            <div>
              {node.name} - {node.age}
            </div>
          );
        }}
      </List>

      {hasPrevious ? (
        <Button onClick={() => loadPrevious(10)}>
          Load more friends
        </Button>
      ) : null}

      {/* Forward pagination controls can go simultaneously here */}
    </>
  );
}
```

* 정방향과 역방향 모두에 대한 API는 정확히 똑같고, 이름만 다릅니다. 정방향으로 페이지네이션할 때는, `after`와 `first` 커넥션 인자가 사용될 것이고, 역방향 페이지네이션에서는 `before`와 `last`가 사용될 것입니다.
* 정방향과 역방향 페이지네이션에 관련된 함수나 변수는 `usePaginationFragment` 호출 한 번에 얻을 수 있습니다. 따라서 정방향과 역방향 페이지네이션은 한 컴포넌트 안에서 동시에 할 수 있습니다.



## 커스텀 커넥션 상태

기본적으로 `usePaginationFragment`와 `@connection`을 사용하면, 릴레이는 새로운 페이지를 정방향 페이지네이션의 경우에는 커넥션의 맨 끝에 붙일(append) 것이고, 역방향 페이지네이션의 경우에는 커넥션의 맨 앞에 붙일(prepend) 것입니다. 이는 컴포넌트가 언제나 지금까지 페이지네이션을 통해 누적된 모든 아이템, 뮤테이션과 구독을 통해 추가되거나 제거된 모든 아이템을 통틀은 전체 커넥션을 렌더링한다는 것을 의미합니다.

하지만 페이지네이션 결과나 커넥션에 대한 업데이트를 합치고 쌓는 방법에 대한 다른 동작 방식이 필요할 수 있습니다. 혹은 커넥션의 변화로부터 로컬 컴포넌트 상태를 끌어내야할 수 있습니다. 다음과 같은 예시가 있을 수 있습니다.

* 커넥션에서 달라지는 *보여줄* 슬라이스나 윈도우를 계속 트래킹하기
* 시각적으로 각 페이지를 분리하기. 이는 가져와지는 각 페이지 속의 정확한 아이템들에 대한 지식을 요구합니다. 
* 한 커넥션의 양쪽 끝 사이(gap)를 계속 추적하면서, 그 양쪽 끝을 동시에 보여주고, 그 사이에 대한 페이지네이션 결과를 계속 합치기. 예를 들어 가장 오래된 댓글이 맨 위에 보여지고, 맨 밑의 섹션에 유저가 추가했거나 실시간 구독에 따라 추가되는 최신 댓글들을 보여주고, 그 사이에 대한 페이지네이션을 지원하는 댓글 리스트를 렌더링하는 상황을 생각해 보세요.


릴레이는 이러한 복잡한 유즈케이스를 더 다룰 수 있도록 계속 작업중입니다. 


> TBD




## 커넥션 새로고침

> TBD




## 커넥션의 페이지를 미리 가져오기

> TBD




## 한 페이지를 한번에 렌더링하기

> TBD



<DocsRating />
