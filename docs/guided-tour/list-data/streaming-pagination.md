---
id: streaming-pagination
title: Streaming Pagination
slug: /guided-tour/list-data/streaming-pagination/
description: Relay guide to streaming pagination
keywords:
- pagination
- usePaginationFragment
- connection
- streaming
---

import DocsRating from '@site/src/core/DocsRating';
import {OssOnly, FbInternalOnly} from 'internaldocs-fb-helpers';

<FbInternalOnly>

추가적으로, `usePaginationFragment`를 릴레이의 [Incremental Data Delivery](../../../guides/incremental-data-delivery/) 기능과 조합할 수 있습니다. 전체 아이템 리스트가 하나의 페이로드에 통째로 담겨서 반환되기를 기다리는 대신에, 커넥션을 가져올 때 각 아이템이 준비되기 시작하면 점진적으로 받기 위해서입니다. 이는 예를 들어 커넥션의 각 아이템을 계산하는 것이 서버 비용이 큰 작업인데, *모든* 아이템이 사용 가능해지는 걸 기다리지 말고 첫 번째 아이템이 준비되면 바로 보여주고 싶을 때 유용할 수 있습니다. 예를 들어 이상적으로 유저가 첫 스토리를 보고, 다른 스토리들은 밑에서 로딩되는 동안 상호작용을 시작할 수 있는 뉴스피드가 있을 것입니다.

</FbInternalOnly>

<OssOnly>

추가적으로, `usePaginationFragment`를 릴레이의 "점진적 데이터 전달" 기능과 조합할 수 있습니다. 전체 아이템 리스트가 하나의 페이로드에 통째로 담겨서 반환되기를 기다리는 대신에, 커넥션을 가져올 때 각 아이템이 준비되기 시작하면 점진적으로 받기 위해서입니다. 이는 예를 들어 커넥션의 각 아이템을 계산하는 것이 서버 비용이 큰 작업인데, *모든* 아이템이 사용 가능해지는 걸 기다리지 말고 첫 번째 아이템이 준비되면 바로 보여주고 싶을 때 유용할 수 있습니다. 예를 들어 이상적으로 유저가 첫 스토리를 보고, 다른 스토리들은 밑에서 로딩되는 동안 상호작용을 시작할 수 있는 뉴스피드가 있을 것입니다.

</OssOnly>

이를 위해서는 `@connection` 지시자 대신에 `@stream_connection` 지시자를 사용하면 됩니다.

```js
import type {FriendsListPaginationQuery} from 'FriendsListPaginationQuery.graphql';
import type {FriendsListComponent_user$key} from 'FriendsList_user.graphql';

const React = require('React');

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
    graphql`
      fragment FriendsListComponent_user on User
      @refetchable(queryName: "FriendsListPaginationQuery") {
        name
        friends(first: $count, after: $cursor)
        @stream_connection(key: "FriendsList_user_friends", initial_count: 2,) {
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

  return (...);
}

module.exports = FriendsListComponent;
```

무슨 일이 일어나는지 봅시다.

* `@stream_connection` 지시자는 `@connection` 지시자 자리에 바로 사용될 수 있습니다. 이는 @connection과 같은 인자를 받으며, 추가로 스트리밍을 제어하기 위한 선택적인 파라미터를 받습니다.
    * `initial_count: Int`: 첫 페이로드에 포함될 아이템 개수를 제어하는 수 (기본값 0). 그 다음의 아이템들은 스트리밍되므로, 0으로 설정되면 첫 리스트는 비고 모든 아이템이 스트리밍될 것입니다. 이 수는 반환될 전체 아이템 수에는 영향을 끼치지 않고, 첫 페이로드에 담길 아이템에 수에만 영향을 끼친다는 것에 주의하세요. 예를 들어, 오늘은 처음에 두 개만 가져오고, 즉시 3개를 더 가져오는 페이지네이션 쿼리를 날리는 프로덕트를 생각해 보세요. 스트리밍을 쓰면, 이 프로덕트는 대신 첫 쿼리에서 아이템 5개를 가져오는데, initial_count=2으로 첫 두 아이템은 빠르게 가져오고, 다음 아이템 3개에 대해 서버에 한번 더 다녀오는 것은 피할 것입니다.
* `usePaginationFragment`의 일반적인 사용처럼, 커넥션은 새 아이템이 서버로부터 스트리밍되면 자동으로 업데이트될 것이고, 컴포넌트는 커넥션의 최신 아이템으로 매번 재렌더링될 것입니다.


<FbInternalOnly>

더 알고 싶다면, [Incremental Data Delivery](../../../guides/incremental-data-delivery/#stream_connection)을 읽어 보세요.

</FbInternalOnly>


<DocsRating />
