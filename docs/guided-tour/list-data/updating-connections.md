---
id: updating-connections
title: 커넥션 업데이트하기
slug: /guided-tour/list-data/updating-connections/
description: Relay guide to updating connections
keywords:
- pagination
- usePaginationFragment
- updating
- connection
---

import DocsRating from '@site/src/core/DocsRating';
import {OssOnly, FbInternalOnly} from 'internaldocs-fb-helpers';

보통 커넥션을 렌더링할 때, 유저 행동에 따라 커넥션에 아이템을 추가하거나 제거하고 싶을 수 있습니다.

[Updating Data](../../updating-data/) 섹션에 설명된 것처럼, 릴레이는 정규화된 GraphQL 데이터에 대한 로컬 인메모리 스토어를 유지합니다. 이 스토어 안에는 레코드가 ID 기준으로 저장됩니다. 뮤테이션, 구독, 로컬 데이터 업데이트가 일어날 때, [`updater`](../../updating-data/graphql-mutations/#updater-functions) 함수를 제공해야만 합니다. 이 함수 안에서는 레코드에 접근하고 읽을 수 있고, 그들에 대해 쓰기와 업데이트도 할 수 있습니다. 레코드가 업데이트되면, 업데이트된 데이터에 영향을 받는 모든 컴포넌트는 알림을 받고 재렌더링될 것입니다.


## 커넥션 레코드

릴레이에서 `@connection` 지시자가 붙은 커넥션 필드는 스토어에 특별한 레코드로 저장되며, 그 커넥션에 대해 지금까지 가져와진 모든 아이템을 쌓아서 들고 있습니다. 커넥션에 아이템을 추가하고 제거하려면 `@connection`을 선언할 때 넘기는 커넥션 키를 사용하여 커넥션 레코드에 접근해야 합니다. 더 자세히 말하자면, 커넥션 키를 가지고 [`updater`](../../updating-data/graphql-mutations/#updater-functions) 함수 안에서 `ConnectionHandler` API를 사용하여 커넥션에 접근할 수 있습니다.

예를 들어, `@connection` 선언을 가지는 다음과 같은 프래그먼트가 주어졌다고 해 봅시다. 이러면 `updater` 함수 안에서 커넥션 레코드에 여러 방법으로 접근할 수 있습니다.

```js
const {graphql} = require('react-relay');

const storyFragment = graphql`
  fragment StoryComponent_story on Story {
    comments @connection(key: "StoryComponent_story_comments_connection") {
      nodes {
        body {
          text
        }
      }
    }
  }
`;
```

### `__id`를 사용하여 커넥션에 접근하기

커넥션의 `__id` 필드를 쿼리할 수 있고, 그를 사용하여 스토어의 레코드에 접근할 수 있습니다.

```js
const fragmentData = useFragment(
  graphql`
    fragment StoryComponent_story on Story {
      comments @connection(key: "StoryComponent_story_comments_connection") {
        # Query for the __id field
        __id

        # ...
      }
    }
  `,
  props.story,
);

// 커넥션 레코드 ID 얻기
const connectionID = fragmentData?.comments?.__id;
```

그를 이용하여 스토어의 레코드에 접근할 수 있습니다.

```js
function updater(store: RecordSourceSelectorProxy) {
  // connectionID는 뮤테이션/구독에 입력으로 넘겨집니다.
  const connection = store.get(connectionID);

  // ...
}
```

:::참고
`__id` 필드는 GraphQL API가 노출해야 하는 것이 **아닙니다**. 이 필드는 릴레이가 자동으로 커넥션 레코드를 식별하기 위해 추가하는 식별자입니다.
:::

### `ConnectionHandler.getConnectionID`를 사용하여 커넥션에 접근하기

만약 커넥션의 부모 레코드 ID를 안다면, `ConnectionHandler.getConnectionID` API를 사용하여 커넥션 레코드에 접근할 수 있습니다.

```js
const {ConnectionHandler} = require('relay-runtime');

function updater(store: RecordSourceSelectorProxy) {
  // Get the connection ID
  const connectionID = ConnectionHandler.getConnectionID(
    storyID, // passed as input to the mutation/subscription
    'StoryComponent_story_comments_connection',
  );

  // Get the connection record
  const connectionRecord = store.get(connectionID);

  // ...
}
```

### `ConnectionHandler.getConnection`을 사용하여 커넥션에 접근하기

커넥션의 부모 레코드에 접근할 수 있다면, `ConnectionHandler.getConnection` API를 사용하여 그 부모를 통해 커넥션에 접근할 수 있습니다.

```js
const {ConnectionHandler} = require('relay-runtime');

function updater(store: RecordSourceSelectorProxy) {
  // Get parent story record
  // storyID is passed as input to the mutation/subscription
  const storyRecord = store.get(storyID);

  // Get the connection record from the parent
  const connectionRecord = ConnectionHandler.getConnection(
    storyRecord,
    'StoryComponent_story_comments_connection',
  );

  // ...
}
```

## 엣지 추가하기

커넥션에 엣지를 추가하는 방법은 여러 개가 있습니다.

### 선언적인 지시자 사용하기

보통, 뮤테이션이나 구독 페이로드는 서버에서 추가된 새로운 엣지 혹은 엣지 리스트를 필드로서 노출할 것입니다. 만약 뮤테이션이나 구독이 엣지 필드를 응답에 노출한다면, 그 필드에 `@appendEdge`와 `@prependEdge`라는 선언전 뮤테이션 지시자를 사용할 수 있습니다. 이를 통해 새로 추가된 엣지를 지정된 커넥션에 추가할 수 있습니다. (이 지시자는 쿼리에 대해서도 동작합니다).

이 대신에, 뮤테이션이나 구독 페이로드는 서버에서 추가된 새로운 노드나 노드의 리스트를 필드로서 노출할 수 있습니다. 만약 뮤테이션이나 구독이 노드나 노드 리스트 필드를 응답에 노출한다면, 그 필드에 `@appendNode` and `@prependNode`라는 선언적 뮤테이션 지시자를 사용할 수 있습니다. 이를 통해 새로 추가된 노드를 엣지로 감싸서 지정된 커넥션에 추가할 수 있습니다. (이 지시자는 쿼리에 대해서도 동작합니다).

These directives accept a `connections` parameter, which needs to be a GraphQL variable containing an array of connection IDs. Connection IDs can be obtained either by using the [`__id` field on connections](#accessing-connections-using-__id) or using the [`ConnectionHandler.getConnectionID`](#accessing-connections-using-connectionhandlergetconnectionid) API.


#### `@appendEdge` / `@prependEdge`

These directives work on a field with a single edge or list of edges. `@prependEdge` will add the selected edges to the beginning of each connection defined in the `connections` array, whereas `@appendEdge` will add the selected edges to the end of each connection in the array.

**Arguments:**
- `connections`: An array of connection IDs. Connection IDs can be obtained either by using the [`__id` field on connections](#accessing-connections-using-__id) or using the [`ConnectionHandler.getConnectionID`](#accessing-connections-using-connectionhandlergetconnectionid) API.


**Example:**

```js
// Get the connection ID using the `__id` field
const connectionID = fragmentData?.comments?.__id;

// Or get it using `ConnectionHandler.getConnectionID()`
const connectionID = ConnectionHandler.getConnectionID(
  '<story-id>',
  'StoryComponent_story_comments_connection',
);

// ...

// Mutation
commitMutation<AppendCommentMutation>(environment, {
  mutation: graphql`
    mutation AppendCommentMutation(
      # Define a GraphQL variable for the connections array
      $connections: [ID!]!
      $input: CommentCreateInput
    ) {
      commentCreate(input: $input) {
        # Use @appendEdge or @prependEdge on the edge field
        feedbackCommentEdge @appendEdge(connections: $connections) {
          cursor
          node {
            id
          }
        }
      }
    }
  `,
  variables: {
    input,
    // Pass the `connections` array
    connections: [connectionID],
  },
});
```


#### `@appendNode` / `@prependNode`

These directives work on a field with a single node or list of nodes, and will create edges with the specified `edgeTypeName`. `@prependNode` will add edges containing the selected nodes to the beginning of each connection defined in the `connections` array, whereas `@appendNode` will add edges containing the selected nodes to the end of each connection in the array.

**Arguments:**
- `connections`: An array of connection IDs. Connection IDs can be obtained either by using the [`__id` field on connections](#accessing-connections-using-__id) or using the [`ConnectionHandler.getConnectionID`](#accessing-connections-using-connectionhandlergetconnectionid) API.
- `edgeTypeName`: The type name of the edge that contains the node, corresponding to the edge type argument in `ConnectionHandler.createEdge`.

**Example:**
```js
// Get the connection ID using the `__id` field
const connectionID = fragmentData?.comments?.__id;

// Or get it using `ConnectionHandler.getConnectionID()`
const connectionID = ConnectionHandler.getConnectionID(
  '<story-id>',
  'StoryComponent_story_comments_connection',
);

// ...

// Mutation
commitMutation<AppendCommentMutation>(environment, {
  mutation: graphql`
    mutation AppendCommentMutation(
      # Define a GraphQL variable for the connections array
      $connections: [ID!]!
      $input: CommentCreateInput
    ) {
      commentCreate(input: $input) {
        # Use @appendNode or @prependNode on the node field
        feedbackCommentNode @appendNode(connections: $connections, edgeTypeName: "CommentsEdge") {
          id
        }
      }
    }
  `,
  variables: {
    input,
    // Pass the `connections` array
    connections: [connectionID],
  },
});
```


#### Order of execution

For all of these directives, they will be executed in the following order within the mutation or subscription, as per the [order of execution of updates](../../updating-data/graphql-mutations/#order-of-execution-of-updater-functions):

* When the mutation is initiated, after the optimistic response is handled, and after the optimistic updater function is executed, the `@prependEdge`, `@appendEdge`, `@prependNode`, and `@appendNode` directives will be applied to the optimistic response.
* If the mutation succeeds, after the data from the network response is merged with the existing values in the store, and after the updater function is executed, the `@prependEdge`, `@appendEdge`, `@prependNode`, and `@appendNode` directives will be applied to the data in the network response.
* If the mutation failed, the updates from processing the `@prependEdge`, `@appendEdge`, `@prependNode`, and `@appendNode` directives will be rolled back.


### Manually adding edges

The directives described [above](#using-declarative-directives) largely remove the need to manually add and remove items from a connection, however, they do not provide as much control as you can get with manually writing an updater, and may not fulfill every use case.

In order to write an updater to modify the connection, we need to make sure we have access to the [connection record](#connection-record). Once we have the connection record, we also need a record for the new edge that we want to add to the connection. Usually, mutation or subscription payloads will contain the new edge that was added; if not, you can also construct a new edge from scratch.

For example, in the following mutation we can query for the newly created edge in the mutation response:

```js
const {graphql} = require('react-relay');

const createCommentMutation = graphql`
  mutation CreateCommentMutation($input: CommentCreateData!) {
    comment_create(input: $input) {
      comment_edge {
        cursor
        node {
          body {
            text
          }
        }
      }
    }
  }
`;
```

* Note that we also query for the `cursor` for the new edge; this isn't strictly necessary, but it is information that will be required if we need to perform pagination based on that `cursor`.


Inside an [`updater`](../../updating-data/graphql-mutations/#updater-functions), we can access the edge inside the mutation response using Relay store APIs:

```js
const {ConnectionHandler} = require('relay-runtime');

function updater(store: RecordSourceSelectorProxy) {
  const storyRecord = store.get(storyID);
  const connectionRecord = ConnectionHandler.getConnection(
    storyRecord,
    'StoryComponent_story_comments_connection',
  );

  // Get the payload returned from the server
  const payload = store.getRootField('comment_create');

  // Get the edge inside the payload
  const serverEdge = payload.getLinkedRecord('comment_edge');

  // Build edge for adding to the connection
  const newEdge = ConnectionHandler.buildConnectionEdge(
    store,
    connectionRecord,
    serverEdge,
  );

  // ...
}
```

* The mutation payload is available as a root field on that store, which can be read using the `store.getRootField` API. In our case, we're reading `comment_create`, which is the root field in the response.
* Note that we need to construct the new edge from the edge received from the server using `ConnectionHandler.buildConnectionEdge` before we can add it to the connection.


If you need to create a new edge from scratch, you can use `ConnectionHandler.createEdge`:

```js
const {ConnectionHandler} = require('relay-runtime');

function updater(store: RecordSourceSelectorProxy) {
  const storyRecord = store.get(storyID);
  const connectionRecord = ConnectionHandler.getConnection(
    storyRecord,
    'StoryComponent_story_comments_connection',
  );

  // Create a new local Comment record
  const id = `client:new_comment:${randomID()}`;
  const newCommentRecord = store.create(id, 'Comment');

  // Create new edge
  const newEdge = ConnectionHandler.createEdge(
    store,
    connectionRecord,
    newCommentRecord,
    'CommentEdge', /* GraphQl Type for edge */
  );

  // ...
}
```


Once we have a new edge record, we can add it to the the connection using `ConnectionHandler.insertEdgeAfter` or `ConnectionHandler.insertEdgeBefore`:

```js
const {ConnectionHandler} = require('relay-runtime');

function updater(store: RecordSourceSelectorProxy) {
  const storyRecord = store.get(storyID);
  const connectionRecord = ConnectionHandler.getConnection(
    storyRecord,
    'StoryComponent_story_comments_connection',
  );

  const newEdge = (...);

  // Add edge to the end of the connection
  ConnectionHandler.insertEdgeAfter(
    connectionRecord,
    newEdge,
  );

  // Add edge to the beginning of the connection
  ConnectionHandler.insertEdgeBefore(
    connectionRecord,
    newEdge,
  );
}
```

* Note that these APIs will *mutate* the connection in place

:::note
Check out our complete [Relay Store APIs](../../../api-reference/store/).
:::

## Removing edges

### Using the declarative deletion directive

Similarly to the [directives to add edges](#using-declarative-directives), we can use the `@deleteEdge` directive to delete edges from connections. If your mutation or subscription exposes a field with the ID or IDs of the nodes that were deleted that you can query for in the response, then you can use the `@deleteEdge` directive on that field to delete the respective edges from the connection (note that this directive also works on queries).

#### `@deleteEdge`

Works on GraphQL fields that return an `ID` or `[ID]`. Will delete the edges with nodes that match the `id` from each connection defined in the `connections` array.

**Arguments:**
- `connections`: An array of connection IDs. Connection IDs can be obtained either by using the [`__id` field on connections](#accessing-connections-using-__id) or using the [`ConnectionHandler.getConnectionID`](#accessing-connections-using-connectionhandlergetconnectionid) API.


**Example:**

```js
// Get the connection ID using the `__id` field
const connectionID = fragmentData?.comments?.__id;

// Or get it using `ConnectionHandler.getConnectionID()`
const connectionID = ConnectionHandler.getConnectionID(
  '<story-id>',
  'StoryComponent_story_comments_connection',
);

// ...

// Mutation
commitMutation<DeleteCommentsMutation>(environment, {
  mutation: graphql`
    mutation DeleteCommentsMutation(
      # Define a GraphQL variable for the connections array
      $connections: [ID!]!
      $input: CommentsDeleteInput
    ) {
      commentsDelete(input: $input) {
        deletedCommentIds @deleteEdge(connections: $connections)
      }
    }
  `,
  variables: {
    input,
    // Pass the `connections` array
    connections: [connectionID],
  },
});
```

### Manually removing edges

`ConnectionHandler` provides a similar API to remove an edge from a connection, via `ConnectionHandler.deleteNode`:

```js
const {ConnectionHandler} = require('RelayModern');

function updater(store: RecordSourceSelectorProxy) {
  const storyRecord = store.get(storyID);
  const connectionRecord = ConnectionHandler.getConnection(
    storyRecord,
    'StoryComponent_story_comments_connection',
  );

  // Remove edge from the connection, given the ID of the node
  ConnectionHandler.deleteNode(
    connectionRecord,
    commentIDToDelete,
  );
}
```

* In this case `ConnectionHandler.deleteNode` will remove an edge given a *`node` ID*. This means it will look up which edge in the connection contains a node with the provided ID, and remove that edge.
* Note that this API will *mutate* the connection in place.


:::note
Remember: when performing any of the operations described here to mutate a connection, any fragment or query components that are rendering the affected connection will be notified and re-render with the latest version of the connection.
:::


## Connection identity with filters

In our previous examples, our connections didn't take any arguments as filters. If you declared a connection that takes arguments as filters, the values used for the filters will be part of the connection identifier. In other words, *each of the values passed in as connection filters will be used to identify the connection in the Relay store.*

:::note
Note that this excludes pagination arguments, i.e. it excludes `first`, `last`, `before`, and `after`.
:::


For example, let's say the `comments` field took the following arguments, which we pass in as GraphQL [variables](../../rendering/variables/):

```js
const {graphql} = require('RelayModern');

const storyFragment = graphql`
  fragment StoryComponent_story on Story {
    comments(
      order_by: $orderBy,
      filter_mode: $filterMode,
      language: $language,
    ) @connection(key: "StoryComponent_story_comments_connection") {
      edges {
        nodes {
          body {
            text
          }
        }
      }
    }
  }
`;
```

In the example above, this means that whatever values we used for `$orderBy`, `$filterMode` and `$language` when we queried for the `comments` field will be part of the connection identifier, and we'll need to use those values when accessing the connection record from the Relay store.

In order to do so, we need to pass a third argument to `ConnectionHandler.getConnection`, with concrete filter values to identify the connection:

```js
const {ConnectionHandler} = require('RelayModern');

function updater(store: RecordSourceSelectorProxy) {
  const storyRecord = store.get(storyID);

  // Get the connection instance for the connection with comments sorted
  // by the date they were added
  const connectionRecordSortedByDate = ConnectionHandler.getConnection(
    storyRecord,
    'StoryComponent_story_comments_connection',
    {order_by: '*DATE_ADDED*', filter_mode: null, language: null}
  );

  // Get the connection instance for the connection that only contains
  // comments made by friends
  const connectionRecordFriendsOnly = ConnectionHandler.getConnection(
    storyRecord,
    'StoryComponent_story_comments_connection',
    {order_by: null, filter_mode: '*FRIENDS_ONLY*', langugage: null}
  );
}
```

This implies that by default, *each combination of values used for filters will produce a different record for the connection.*

When making updates to a connection, you will need to make sure to update all of the relevant records affected by a change. For example, if we were to add a new comment to our example connection, we'd need to make sure *not* to add the comment to the `FRIENDS_ONLY` connection, if the new comment wasn't made by a friend of the user:

```js
const {ConnectionHandler} = require('relay-runtime');

function updater(store: RecordSourceSelectorProxy) {
  const storyRecord = store.get(storyID);

  // Get the connection instance for the connection with comments sorted
  // by the date they were added
  const connectionRecordSortedByDate = ConnectionHandler.getConnection(
    storyRecord,
    'StoryComponent_story_comments_connection',
    {order_by: '*DATE_ADDED*', filter_mode: null, language: null}
  );

  // Get the connection instance for the connection that only contains
  // comments made by friends
  const connectionRecordFriendsOnly = ConnectionHandler.getConnection(
    storyRecord,
    'StoryComponent_story_comments_connection',
    {order_by: null, filter_mode: '*FRIENDS_ONLY*', language: null}
  );

  const newComment = (...);
  const newEdge = (...);

  ConnectionHandler.insertEdgeAfter(
    connectionRecordSortedByDate,
    newEdge,
  );

  if (isMadeByFriend(storyRecord, newComment) {
    // Only add new comment to friends-only connection if the comment
    // was made by a friend
    ConnectionHandler.insertEdgeAfter(
      connectionRecordFriendsOnly,
      newEdge,
    );
  }
}
```



_Managing connections with many filters:_

As you can see, just adding a few filters to a connection can make the complexity and number of connection records that need to be managed explode. In order to more easily manage this, Relay provides 2 strategies:

1) Specify exactly *which* filters should be used as connection identifiers.

By default, *all* non-pagination filters will be used as part of the connection identifier. However, when declaring a `@connection`, you can specify the exact set of filters to use for connection identity:

```js
const {graphql} = require('relay-runtime');

const storyFragment = graphql`
  fragment StoryComponent_story on Story {
    comments(
      order_by: $orderBy
      filter_mode: $filterMode
      language: $language
    )
      @connection(
        key: "StoryComponent_story_comments_connection"
        filters: ["order_by", "filter_mode"]
      ) {
      edges {
        nodes {
          body {
            text
          }
        }
      }
    }
  }
`;
```

* By specifying `filters` when declaring the `@connection`, we're indicating to Relay the exact set of filter values that should be used as part of connection identity. In this case, we're excluding `language`, which means that only values for `order_by` and `filter_mode` will affect connection identity and thus produce new connection records.
* Conceptually, this means that we're specifying which arguments affect the output of the connection from the server, or in other words, which arguments are *actually* *filters*. If one of the connection arguments doesn't actually change the set of items that are returned from the server, or their ordering, then it isn't really a filter on the connection, and we don't need to identify the connection differently when that value changes. In our example, changing the `language` of the comments we request doesn't change the set of comments that are returned by the connection, so it is safe to exclude it from `filters`.
* This can also be useful if we know that any of the connection arguments will never change in our app, in which case it would also be safe to exclude from `filters`.



2) An easier API alternative to manage multiple connections with multiple filter values is still pending


> TBD



<DocsRating />
