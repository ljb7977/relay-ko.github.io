---
id: refreshing-queries
title: Refreshing Queries
slug: /guided-tour/refetching/refreshing-queries/
description: Relay guide to refreshing queries
keywords:
- refreshing
- queries
---

import DocsRating from '@site/src/core/DocsRating';
import {OssOnly, FbInternalOnly} from 'internaldocs-fb-helpers';
import FbRefreshingUsingRealTimeFeatures from './fb/FbRefreshingUsingRealTimeFeatures.md';
import FbRefreshingQueriesUsingUseQueryLoader from './fb/FbRefreshingQueriesUsingUseQueryLoader.md';
import FbAvoidSuspenseCaution from './fb/FbAvoidSuspenseCaution.md';
import FbRefreshingQueriesUsingUseLazyLoadQuery from './fb/FbRefreshingQueriesUsingUseLazyLoadQuery.md';
import OssAvoidSuspenseNote from './OssAvoidSuspenseNote.md';


**"쿼리 새로고침"**이라고 하면, 쿼리에 의해 원래 렌더링됐던 바로 그 데이터를 다시 가져오는 것을 의미합니다. 이는 서버로부터 가장 최신 버전의 데이터를 가져오기 위함입니다.

## 실시간 기능 사용하기

<FbInternalOnly>
  <FbRefreshingUsingRealTimeFeatures />
</FbInternalOnly>

<OssOnly>
우리가 우리의 데이터를 서버의 최근 버전을 이용하여 최신으로 유지하기를 원한다면, 가장 처음으로 생각해야 할 것은 실시간 기능을 사용하기에 적합한지입니다. 실시간 기능은 수동으로 데이터를 주기적으로 새로고침할 필요 없이, 자동으로 데이터를 최신으로 유지하는 것을 더 쉽게 만들어 줍니다.

이것의 한 예는 [GraphQL Subscriptions](https://relay.dev/docs/guided-tour/updating-data/graphql-subscriptions)를 사용하는 것입니다. 구독은 서버와 [network layer](https://relay.dev/docs/guided-tour/updating-data/graphql-subscriptions/#configuring-the-network-layer)에 추가적인 설정을 필요로 할 것입니다.
</OssOnly>

## `useQueryLoader` / `loadQuery`를 사용하는 경우

[Fetching Queries for Render](../../rendering/queries/#fetching-queries-for-render) 섹션에서 설명된 [`useQueryLoader`](../../../api-reference/use-query-loader/) 훅을 이용하여 쿼리를 새로고침하기 위해서는, `loadQuery`를 다시 부르기만 하면 됩니다.

<FbInternalOnly>
  <FbRefreshingQueriesUsingUseQueryLoader />
</FbInternalOnly>

<OssOnly>

```js
/**
 * App.react.js
 */
import type {AppQuery as AppQueryType} from 'AppQuery.graphql';

const AppQuery = require('__generated__/AppQuery.graphql');

function App(props: Props) {
  const [queryRef, loadQuery] = useQueryLoader<AppQueryType>(
    AppQuery,
    props.appQueryRef /* initial query ref */
  );

  const refresh = useCallback(() => {
    // Load the query again using the same original variables.
    // Calling loadQuery will update the value of queryRef.
    // The fetchPolicy ensures we always fetch from the server and skip
    // the local data cache.
    const {variables} = props.appQueryRef;
    loadQuery(variables, {fetchPolicy: 'network-only'});
  }, [/* ... */]);

  return (
    <React.Suspense fallback="Loading query...">
      <MainContent
        refresh={refresh}
        queryRef={queryRef}
      />
    </React.Suspense>
  );
}
```

```js
/**
 * MainContent.react.js
 */
import type {AppQuery as AppQueryType} from 'AppQuery.graphql';

// Renders the preloaded query, given the query reference
function MainContent(props) {
  const {refresh, queryRef} = props;
  const data = usePreloadedQuery<AppQueryType>(
    graphql`
      query AppQuery($id: ID!) {
        user(id: $id) {
          name
          friends {
            count
          }
        }
      }
    `,
    queryRef,
  );

  return (
    <>
      <h1>{data.user?.name}</h1>
      <div>Friends count: {data.user.friends?.count}</div>
      <Button
        onClick={() => refresh()}>
        Fetch latest count
      </Button>
    </>
  );
}
```

여기서 무슨 일이 일어나는지 분석해 봅시다.

* 새로고침을 위해 이벤트 핸들러의 `loadQuery`를 불렀습니다. 따라서 네트워크 요청은 즉시 시작되고, 업데이트된 `queryRef`를 `usePreloadedQuery`를 사용하는 `MainContent` 컴포넌트에 전달합니다. 그리고 MainContent 컴포넌트가 업데이트된 데이터를 렌더링합니다.
* 언제나 로컬 데이터 캐시를 스킵하고 데이터를 네트워크로부터 가져오도록, `fetchPolicy`의 값을 `'network-only'`로 설정합니다.
* `loadQuery` 호출은 컴포넌트를 재렌더링하고 `usePreloadedQuery`의 중단(suspend)을 발생시킬 수 있습니다 ([Loading States with Suspense](../../rendering/loading-states/)에 설명된 것처럼). `fetchPolicy` 때문에 네트워크 요청이 언제나 발생할 것이기 때문입니다. 따라서 폴백(fallback) 로딩 상태를 보여주기 위해 `MainContent`를 감싸는 `Suspense` 바운더리를 꼭 만들어 주어야 합니다.

</OssOnly>

### Suspense를 피해야 한다면

몇몇 경우, Suspense 폴백이 이미 렌더링된 내용을 가릴 수 있기 때문에 그를 보여주는 것을 피해야 할 수 있습니다. 그런 경우, [`fetchQuery`](../../../api-reference/fetch-query/)를 대신 사용하고 로딩 상태를 수동으로 추적하면 됩니다.

<FbInternalOnly>
  <FbAvoidSuspenseCaution />
</FbInternalOnly>

<OssOnly>
  <OssAvoidSuspenseNote />
</OssOnly>

```js
/**
 * App.react.js
 */
import type {AppQuery as AppQueryType} from 'AppQuery.graphql';

const AppQuery = require('__generated__/AppQuery.graphql');

function App(props: Props) {
  const environment = useRelayEnvironment();
  const [queryRef, loadQuery] = useQueryLoader<AppQueryType>(
    AppQuery,
    props.appQueryRef /* initial query ref */
  );
  const [isRefreshing, setIsRefreshing] = useState(false)

  const refresh = useCallback(() => {
    if (isRefreshing) { return; }
    const {variables} = props.appQueryRef;
    setIsRefreshing(true);

    // fetchQuery will fetch the query and write
    // the data to the Relay store. This will ensure
    // that when we re-render, the data is already
    // cached and we don't suspend
    fetchQuery(environment, AppQuery, variables)
      .subscribe({
        complete: () => {
          setIsRefreshing(false);

          // *After* the query has been fetched, we call
          // loadQuery again to re-render with a new
          // queryRef.
          // At this point the data for the query should
          // be cached, so we use the 'store-only'
          // fetchPolicy to avoid suspending.
          loadQuery(variables, {fetchPolicy: 'store-only'});
        }
        error: () => {
          setIsRefreshing(false);
        }
      });
  }, [/* ... */]);

  return (
    <React.Suspense fallback="Loading query...">
      <MainContent
        isRefreshing={isRefreshing}
        refresh={refresh}
        queryRef={queryRef}
      />
    </React.Suspense>
  );
}
```

무슨 일이 일어나는지 봅시다.

* 새로고침할 때, suspend를 피해야 하기 때문에 우리만의 `isRefreshing` 로딩 상태를 추적합니다. 우리는 이 상태를 사용하여 `MainContent`를 숨기지 않고 `MainContent` 컴포넌트 안에서 busy spinner 혹은 그와 비슷한 로딩 UI를 렌더링할 수 있습니다.
* 이벤트 핸들러에서 먼저 `fetchQuery`를 호출합니다. 이는 쿼리 결과를 가져와서 데이터를 로컬 릴레이 스토어에 쓸 것입니다. `fetchQuery` 네트워크 요청이 완료되면, `loadQuery`를 불러서 업데이트된 `queryRef`를 얻고, 이를 `usePreloadedQuery`에 넘겨서 업데이트된 데이터를 렌더링합니다.
* 이 시점에서, `loadQuery`가 호출되었을 때, 쿼리에 관한 데이터는 이미 로컬 릴레이 스토어에 캐시되어 있어야 합니다. 그래야   `fetchPolicy`를 `'store-only'`로 설정하여 중단을 피하고, 이미 캐시된 데이터만을 사용할 수 있기 때문입니다.


## `useLazyLoadQuery`를 사용하는 경우

[Lazily Fetching Queries during Render](../../rendering/queries/#lazily-fetching-queries-during-render) 섹션에 설명된 [`useLazyLoadQuery`](../../../api-reference/use-lazy-load-query/) 훅을 사용하여 쿼리를 새로고침하려면, 다음과 같이 하면 됩니다.

<FbInternalOnly>
  <FbRefreshingQueriesUsingUseLazyLoadQuery />
</FbInternalOnly>

<OssOnly>

```js
/**
 * App.react.js
 */
import type {AppQuery as AppQueryType} from 'AppQuery.graphql';

const AppQuery = require('__generated__/AppQuery.graphql');

function App(props: Props) {
  const variables = {id: '4'};
  const [refreshedQueryOptions, setRefreshedQueryOptions] = useState(null);

  const refresh = useCallback(() => {
    // Trigger a re-render of useLazyLoadQuery with the same variables,
    // but an updated fetchKey and fetchPolicy.
    // The new fetchKey will ensure that the query is fully
    // re-evaluated and refetched.
    // The fetchPolicy ensures that we always fetch from the network
    // and skip the local data cache.
    setRefreshedQueryOptions(prev => ({
      fetchKey: (prev?.fetchKey ?? 0) + 1,
      fetchPolicy: 'network-only',
    }));
  }, [/* ... */]);

  return (
    <React.Suspense fallback="Loading query...">
      <MainContent
        refresh={refresh}
        queryOptions={refreshedQueryOptions ?? {}}
        variables={variables}
      />
    </React.Suspense>
  );
```

```js
/**
 * MainContent.react.js
 */

// Fetches and renders the query, given the fetch options
function MainContent(props) {
  const {refresh, queryOptions, variables} = props;
  const data = useLazyLoadQuery(
    graphql`
      query AppQuery($id: ID!) {
        user(id: $id) {
          name
          friends {
            count
          }
        }
      }
    `,
    variables,
    queryOptions,
  );

  return (
    <>
      <h1>{data.user?.name}</h1>
      <div>Friends count: {data.user.friends?.count}</div>
      <Button
        onClick={() => refresh()}>
        Fetch latest count
      </Button>
    </>
  );
}
```

무슨 일이 일어나는지 봅시다

* 우리는 상태의 새 옵션을 설정함으로써, 컴포넌트를 이벤트 핸들러 안에서 업데이트하여 새로고침합니다. 이는 `useLazyLoadQuery`를 사용하는 `MainContent` 컴포넌트를 새로운 `fetchKey`와 `fetchPolicy`을 가지고 재렌더링합니다. 또한 렌더링하면서 쿼리를 다시 가져옵니다.
* We are passing a new value of `fetchKey` which we increment on every update. Passing a new `fetchKey` to `useLazyLoadQuery` on every update will ensure that the query is fully re-evaluated and refetched.
* We are passing a `fetchPolicy` of `'network-only'` to ensure that we always fetch from the network and skip the local data cache.
* The state update in `refresh` will cause the component to suspend (as explained in [Loading States with Suspense](../../rendering/loading-states/)), since a network request will always be made due to the `fetchPolicy` we are using. This means that we'll need to make sure that there's a `Suspense` boundary wrapping the `MainContent` component in order to show a fallback loading state.

</OssOnly>

### If you need to avoid Suspense

In some cases, you might want to avoid showing a Suspense fallback, which would hide the already rendered content. For these cases, you can use [`fetchQuery`](../../../api-reference/fetch-query/) instead, and manually keep track of a loading state:

<FbInternalOnly>
  <FbAvoidSuspenseCaution />
</FbInternalOnly>

<OssOnly>
  <OssAvoidSuspenseNote />
</OssOnly>

```js
/**
 * App.react.js
 */
import type {AppQuery as AppQueryType} from 'AppQuery.graphql';

const AppQuery = require('__generated__/AppQuery.graphql');

function App(props: Props) {
  const variables = {id: '4'}
  const environment = useRelayEnvironment();
  const [refreshedQueryOptions, setRefreshedQueryOptions] = useState(null);
  const [isRefreshing, setIsRefreshing] = useState(false)

  const refresh = useCallback(() => {
    if (isRefreshing) { return; }
    setIsRefreshing(true);

    // fetchQuery will fetch the query and write
    // the data to the Relay store. This will ensure
    // that when we re-render, the data is already
    // cached and we don't suspend
    fetchQuery(environment, AppQuery, variables)
      .subscribe({
        complete: () => {
          setIsRefreshing(false);

          // *After* the query has been fetched, we update
          // our state to re-render with the new fetchKey
          // and fetchPolicy.
          // At this point the data for the query should
          // be cached, so we use the 'store-only'
          // fetchPolicy to avoid suspending.
          setRefreshedQueryOptions(prev => ({
            fetchKey: (prev?.fetchKey ?? 0) + 1,
            fetchPolicy: 'store-only',
          }));
        }
        error: () => {
          setIsRefreshing(false);
        }
      });
  }, [/* ... */]);

  return (
    <React.Suspense fallback="Loading query...">
      <MainContent
        isRefreshing={isRefreshing}
        refresh={refresh}
        queryOptions={refreshedQueryOptions ?? {}}
        variables={variables}
      />
    </React.Suspense>
  );
}
```

Let's distill what's going on here:

* When refreshing, we now keep track of our own `isRefreshing` loading state, since we are avoiding suspending. We can use this state to render a busy spinner or similar loading UI inside the `MainContent` component, *without* hiding the `MainContent`.
* In the event handler, we first call `fetchQuery`, which will fetch the query and write the data to the local Relay store. When the `fetchQuery` network request completes, we update our state so that we re-render an updated `fetchKey` and `fetchPolicy` that we then pass to `useLazyLoadQuery` in order render the updated data, similar to the previous example.
* At this point, when we update the state, the data for the query should already be cached in the local Relay store, so we use `fetchPolicy` of `'store-only'` to avoid suspending and only read the already cached data.

<DocsRating />
