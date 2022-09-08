---
id: refetching-connections
title: Refetching Connections (Using and Changing Filters)
slug: /guided-tour/list-data/refetching-connections/
description: Relay guide to refetching connections
keywords:
- pagination
- refetching
- connection
- useRefetchableFragment
---

import DocsRating from '@site/src/core/DocsRating';
import {OssOnly, FbInternalOnly} from 'internaldocs-fb-helpers';
import FbRefetchingConnectionsUsingUseTransition from './fb/FbRefetchingConnectionsUsingUseTransition.md';

보통 데이터의 리스트를 쿼리할 때, 쿼리에 다른 값을 전달하여 필터로 사용할 수 있습니다. 필터를 통해 쿼리의 결과를 바꾸거나, 정렬 방식을 바꿀 수 있습니다.

그러한 예시들은 다음과 같습니다. 

* 검색 자동 완성 만들기. 유저가 입력한 검색어로 필터링된 결과가 내려올 것입니다.
* 포스트에 대해 현재 보여주는 댓글의 정렬 모드 변경하기. 서버로부터 완전히 다른 결과가 내려올 수 있습니다.
* 뉴스피드의 랭킹과 정렬 방식 바꾸기


구체적으로 말하면, GraphQL의 커넥션 필드는 쿼리 결과를 필터링하거나 정렬하기 위한 인자를 받을 수 있습니다.

```graphql
fragment UserFragment on User {
  name
  friends(order_by: DATE_ADDED, search_term: "Alice", first: 10) {
    edges {
      node {
        name
        age
      }
    }
  }
}
```


릴레이에서는 GraphQL [variables](../../rendering/variables/)를 이용하여 그러한 인자를 평소처럼 넘길 수 있습니다.

```js
type Props = {
  userRef: FriendsListComponent_user$key,
};

function FriendsListComponent(props: Props) {
  const userRef = props.userRef;

  const {data, ...} = usePaginationFragment(
    graphql`
      fragment FriendsListComponent_user on User {
        name
        friends(
          order_by: $orderBy,
          search_term: $searchTerm,
          after: $cursor,
          first: $count,
        ) @connection(key: "FriendsListComponent_user_friends_connection") {
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

  return (...);
}
```


페이지네이션하는 동안 그런 필터값은 계속 같게 유지됩니다.

```js
type Props = {
  userRef: FriendsListComponent_user$key,
};

function FriendsListComponent(props: Props) {
  const userRef = props.userRef;

  const {data, loadNext} = usePaginationFragment(
    graphql`
      fragment FriendsListComponent_user on User {
        name
        friends(order_by: $orderBy, search_term: $searchTerm)
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
      <List items={data.friends?.nodes}>{...}</List>

      {/*
       Loading the next items will use the original order_by and search_term
       values used for the initial query
      */ }
      <Button onClick={() => loadNext(10)}>Load more friends</Button>
    </>
  );
}
```
* `loadNext`를 호출하면 첫 쿼리에 사용된 `order_by`와 `search_term` 값이 계속 사용된다는 것을 알아두세요. 페이지네이션하는 동안 이 값들은 바뀌지 않을 것이고, 바뀌어서도 *안 됩니다*.

만약 커넥션을 리페치할 때 다른 변수를 쓰고 싶다면, [Refetching Fragments with Different Data](../../refetching/refetching-fragments-with-different-data/)에서 하는 것처럼 `usePaginationFragment`가 제공하는 `refetch` 함수를 사용할 수 있습니다.

<FbInternalOnly>
  <FbRefetchingConnectionsUsingUseTransition />
</FbInternalOnly>

<OssOnly>

```js
/**
 * FriendsListComponent.react.js
 */
import type {FriendsListComponent_user$key} from 'FriendsListComponent_user.graphql';

const React = require('React');
const {useState, useEffect} = require('React');

const {graphql, usePaginationFragment} = require('react-relay');


type Props = {
  searchTerm?: string,
  user: FriendsListComponent_user$key,
};

function FriendsListComponent(props: Props) {
  const searchTerm = props.searchTerm;
  const {data, loadNext, refetch} = usePaginationFragment(
    graphql`
      fragment FriendsListComponent_user on User {
        name
        friends(
          order_by: $orderBy,
          search_term: $searchTerm,
          after: $cursor,
          first: $count,
        ) @connection(key: "FriendsListComponent_user_friends_connection") {
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

  useEffect(() => {
    // When the searchTerm provided via props changes, refetch the connection
    // with the new searchTerm
    refetch({first: 10, search_term: searchTerm}, {fetchPolicy: 'store-or-network'});
  }, [searchTerm])

  return (
    <>
      <h1>Friends of {data.name}:</h1>

      {/* When the button is clicked, refetch the connection but sorted differently */}
      <Button
        onClick={() =>
          refetch({first: 10, orderBy: 'DATE_ADDED'});
        }>
        Sort by date added
      </Button>

      <List items={data.friends?.nodes}>...</List>
      <Button onClick={() => loadNext(10)}>Load more friends</Button>
    </>
  );
}
```

무슨 일이 일어나고 있는지 살펴봅시다.

* `refetch`를 부르고 새로운 변수들을 넘기면 *그 새로운 변수들로* 프래그먼트를 다시 가져올 것입니다. 이때는 자동 생성된 쿼리가 예상하는 변수들의 부분집합을 넘겨야 합니다. 프래그먼트의 타입이 `id` 필드를 가진다면, 생성된 쿼리는 `id`와, 프래그먼트로부터 이행적으로 참조된 다른 어떤 변수들도 요구할 것입니다.
    * 위의 예시에서는 가져오고 싶은 데이터의 개수를 `first`로 넘겨야 하고, `orderBy`나 `searchTerm`과 같은 필터에는 다른 값을 넣어도 됩니다.
* 이것은 컴포넌트를 재렌더링할 것이고, 네트워크 요청을 보내고 기다려야 한다면 ([Loading States with Suspense](../../rendering/loading-states/)에 설명된 것처럼) 그 컴포넌트를 중단시킬 수도 있습니다. 만약 `refetch` 가 컴포넌트 중단을 유발한다면, 이 컴포넌트를 `Suspense` 바운더리로 반드시 위에서 감싸야 합니다.
* 개념적으로, refetch를 부르는 것은 커넥션을 *처음부터* 가져오는 것입니다. 다시 말해 커넥션을 다시 맨 앞에서부터 가져오고 페이지네이션 상태를 리셋하는 것입니다. 예를 들어 커넥션을 다른 `search_term`을 써서 가져온다면, 이전의 `search_term`에 대한 페이지네이션 정보는 더이상 의미가 없습니다. 근본적으로 새로운 아이템 리스트에 대해 페이지네이션하는 것이기 때문입니다.

</OssOnly>




<DocsRating />
