---
id: rendering-connections
title: Rendering Connections
slug: /guided-tour/list-data/rendering-connections/
description: Relay guide to rendering connections
keywords:
- pagination
- usePaginationFragment
- connection
---

import DocsRating from '@site/src/core/DocsRating';
import FbSuspenseListAlternative from './fb/FbSuspenseListAlternative.md';
import FbRenderingConnectionsUsingSuspenseList from './fb/FbRenderingConnectionsUsingSuspenseList.md';
import {OssOnly, FbInternalOnly} from 'internaldocs-fb-helpers';

릴레이에서 GraphQL 커넥션으로 된 데이터의 리스트를 출력하려면, 커넥션을 쿼리하는 프래그먼트를 선언해야 합니다.

```js
const {graphql} = require('RelayModern');

const userFragment = graphql`
  fragment UserFragment on User {
    name
    friends(after: $cursor, first: $count)
      @connection(key: "UserFragment_friends") {
      edges {
        node {
          ...FriendComponent
        }
      }
    }
  }
`;
```

* 위의 예시는 커넥션인 `friends` 필드를 쿼리하고 있습니다. 다르게 말하면, `friends` 필드는 커넥션 명세를 준수합니다. 구체적으로, 우리는 커넥션 안의 엣지(`edges`)와 노드(`node`)들을 쿼리할 수 있습니다. 보통 `edges`는 엔티티 간의 관계 정보를 포함하고, `node`는 그 관계의 양 끝에 있는 실제 엔티티를 뜻합니다. 위 예시에서 `node`는 유저의 친구들을 나타내는 `User` 타입의 객체입니다.
* 이 커넥션에 대해 페이지네이션을 하고 싶다는 것을 릴레이에게 지시하기 위해 `@connection` 지시자로 필드를 마킹해야 합니다. 또한 이 커넥션에게 `key`라는 정적인 고유 식별자가 주어져야 합니다. 이 커넥션 키의 이름은 `<fragment_name>_<field_name>`의 형태를 추천합니다.
* 나중의 [Updating Connections](../updating-connections/) 섹션에서는 필드를 @connection으로 마킹하고 고유한 `key`를 주어야 하는 이유를 더 깊게 알아보겠습니다.


커넥션을 쿼리하는 이 프래그먼트를 렌더링하기 위해서는 `usePaginationFragment` 훅을 사용할 수 있습니다.

<FbInternalOnly>
  <FbRenderingConnectionsUsingSuspenseList />
</FbInternalOnly>

<OssOnly>

```js
import type {FriendsListPaginationQuery} from 'FriendsListPaginationQuery.graphql';
import type {FriendsListComponent_user$key} from 'FriendsList_user.graphql';

const React = require('React');
const {Suspense} = require('React');

const {graphql, usePaginationFragment} = require('react-relay');

type Props = {
  user: FriendsListComponent_user$key,
};

function FriendsListComponent(props: Props) {
  const {data} = usePaginationFragment<FriendsListPaginationQuery, _>(
    graphql`
      fragment FriendsListComponent_user on User
      @refetchable(queryName: "FriendsListPaginationQuery") {
        name
        friends(first: $count, after: $cursor)
        @connection(key: "FriendsList_user_friends") {
          edges {
            node {
              ...FriendComponent
            }
          }
        }
      }
    `,
    props.user,
  );


  return (
    <>
      {data.name != null ? <h1>Friends of {data.name}:</h1> : null}

      <div>
        {/* Extract each friend from the resulting data */}
        {(data.friends?.edges ?? []).map(edge => {
          const node = edge.node;
          return (
            <Suspense fallback={<Glimmer />}>
              <FriendComponent user={node} />
            </Suspense>
          );
        })}
      </div>
    </>
  );
}

module.exports = FriendsListComponent;
```
<FbSuspenseListAlternative />

* `usePaginationFragment`은 `useFragment`와 같은 방식으로 동작하기 때문에 ([Fragments](../../rendering/fragments/) 섹션 참조), 프래그먼트에 의해 정의된 대로 친구 목록은 `data.friends.edges.node`을 통해 얻을 수 있습니다. 하지만, 몇몇 추가 사항이 있습니다.
    * 프래그먼트의 커넥션 필드에 `@connection` 지시자가 붙어 있어야 합니다.
    * 프래그먼트에 `@refetchable` 지시자가 붙어 있어야 합니다. `@refetchable` 지시자는 "다시 가져올 수 있는 (refetchable)" 프래그먼트에만 붙을 수 있는 것에 주의하세요. 즉,`Viewer`, `Query`, `Node`를 구현하는 (즉, `id` 필드가 있는 타입) 타입, 혹은 `@fetchable` 타입 위의 프래그먼트여야 합니다. <FbInternalOnly> For more info on `@fetchable` types, see [this post](https://fb.workplace.com/groups/graphql.fyi/permalink/1539541276187011/). </FbInternalOnly>
* `usePaginationFragment`는 두 개의 Flow 타입 파라미터를 받습니다. 첫 번째 파라미터에는 생성된 쿼리의 타입을(예시에서는 `FriendsListPaginationQuery`) 넘기고, 두 번째 타입은 언제나 추론될 수 있으므로 단지 `_`를 넘기면 됩니다.

</OssOnly>

<DocsRating />
