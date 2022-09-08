---
id: pagination
title: Pagination
slug: /guided-tour/list-data/pagination/
description: Relay guide to pagination
keywords:
- pagination
- usePaginationFragment
---

import DocsRating from '@site/src/core/DocsRating';
import {OssOnly, FbInternalOnly} from 'internaldocs-fb-helpers';
import FbPaginationUsingUseTransition from './fb/FbPaginationUsingUseTransition.md';

커넥션에 대해 페이지네이션을 실제로 수행하려면, `usePaginationFragment`가 제공하는 `loadNext` 함수를 써서 다음 페이지를 가져와야 합니다.

<FbInternalOnly>
  <FbPaginationUsingUseTransition />
</FbInternalOnly>

<OssOnly>

```js
import type {FriendsListPaginationQuery} from 'FriendsListPaginationQuery.graphql';
import type {FriendsListComponent_user$key} from 'FriendsList_user.graphql';

const React = require('React');

const {graphql, usePaginationFragment} = require('react-relay');

const {Suspense} = require('React');

type Props = {
  user: FriendsListComponent_user$key,
};

function FriendsListComponent(props: Props) {
  const {data, loadNext} = usePaginationFragment<FriendsListPaginationQuery, _>(
    graphql`
      fragment FriendsListComponent_user on User
      @refetchable(queryName: "FriendsListPaginationQuery") {
        name
        friends(first: $count, after: $cursor)
        @connection(key: "FriendsList_user_friends") {
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

  return (
    <>
      <h1>Friends of {data.name}:</h1>
      <div>
        {(data.friends?.edges ?? []).map(edge => {
          const node = edge.node;
          return (
            <Suspense fallback={<Glimmer />}>
              <FriendComponent user={node} />
            </Suspense>
          );
        })}
      </div>

      <Button
        onClick={() => {
          loadNext(10)
        }}>
        Load more friends
      </Button>
    </>
  );
}

module.exports = FriendsListComponent;
```

여기서 무엇이 일어나는지를 분석해 봅시다.

* `loadNext`는 커넥션에서 더 가져올 아이템 개수를 받습니다. 위의 예시에서는 `loadNext`를 부르면 현재 렌더링된 `User`의 친구 목록에서 다음 10명의 친구를 가져올 것입니다.
* 다음 아이템들을 가져오는 요청이 완료되면, 커넥션 변수는 자동으로 업데이트되고 컴포넌트는 그 커넥션의 최신 아이템들을 가지고 재렌더링될 것입니다.  이는 위 예시의 `friends` 필드가 언제나 지금까지 가져온 *모든* 친구들을 담고 있다는 것을 의미합니다. 기본적으로, *릴레이는 페이지네이션 요청을 완료하면 자동으로 새 아이템들을 커넥션에 추가하고,* 프래그먼트 컴포넌트에서 사용할 수 있도록 만듭니다. 다른 동작이 필요하다면, [Advanced Pagination Use Cases](../advanced-pagination/) 섹션을 참고하세요.
* `loadNext`는 자신이 불린 컴포넌트나 새 자식 컴포넌트들을 중단시킬(suspend) 수 있습니다 ([Loading States with Suspense](../../rendering/loading-states/)에서 설명된 것처럼). 이는 이 컴포넌트를 위에서 감싸는 `Suspense` 바운더리를 꼭 만들어야 한다는 것을 의미합니다.

</OssOnly>


또한 보통 여기서 더 가져올 아이템이 있는지 알고 싶을 것입니다. 이를 위해서는 `hasNext` 값을 사용할 수 있습니다. 이 값 또한 `usePaginationFragment`에서 얻을 수 있습니다.

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
  // ...
  const {
    data,
    loadNext,
    hasNext,
  } = usePaginationFragment<FriendsListPaginationQuery, _>(
    graphql`...`,
    props.user,
  );

  return (
    <>
      <h1>Friends of {data.name}:</h1>
      {/* ... */}

      {/* Only render button if there are more friends to load in the list */}
      {hasNext ? (
        <Button
          onClick={/* ... */}>
          Load more friends
        </Button>
      ) : null}
    </>
  );
}

module.exports = FriendsListComponent;
```

* `hasNext`는 커넥션에서 가져올 수 있는 아이템이 더 있는지를 나타내는 참/거짓 값입니다. 이 값은 다른 UI 컨트롤이 렌더링되어야 하는지를 결정하는 데에도 유용할 수 있습니다. 위의 예시에서는 커넥션에 친구가 더 있다면 `Button`을 렌더링합니다.



<DocsRating />
