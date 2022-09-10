---
id: entrypoint
title: Entrypoint
sidebar_label: Entrypoint
---

> Entrypoint is a pattern in Relay that enables the [render-as-you-fetch](https://17.reactjs.org/docs/concurrent-mode-suspense.html#approach-3-render-as-you-fetch-using-suspense) pattern to be used in a simple way.

> It encapsulates a react component and its data requirements (queries fragment) together in a way that allows for parallel data fetching.

The under example is from [relay-times](https://github.com/dohaicuong/relay-times/tree/master/relay-entrypoint)

1. setup relay persisted_query
```json
{
  "src": ".",
  "schema": "./schema.graphql",
  "excludes": ["**/dist/**", "**/node_modules/**", "**/__generated__/**"],
  "language": "typescript",
  "eagerEsModules": true,
  "persistConfig": {
    "file": "./persisted_queries.json",
    "algorithm": "MD5"
  }
}
```

2. page with usePreloadedQuery
```tsx
export type PostProps = {
  queries: {
    postQueryRef: PreloadedQuery<PostQuery>
  }
}

const Post: React.FC<PostProps> = ({ queries }) => {
  const data = usePreloadedQuery<PostQuery>(
    graphql`
      query PostQuery($id: ID!) @preloadable {
        node(id: $id) {
          ... on Post {
            id
            title
            body
          }
        }
      }
    `,
    queries.postQueryRef
  )


  return (
    <Stack>
      <Title order={1}>{data.node?.title}</Title>
      <Text>{data.node?.body}</Text>
    </Stack>
  )
}
```

3. setup entrypoint (need to use jsresource)
```ts
export type PostEntrypointParams = {
  id: string
}

const PostEntrypoint: EntryPoint<any, PostEntrypointParams> = {
  root: JSResource('Post', () => import(/* webpackPrefetch: true */ '../features/post/Post')) as any,
  getPreloadProps: params => ({
    queries: {
      postQueryRef: {
        parameters: {
          kind: 'PreloadableConcreteRequest',
          params: PostQuery.params
        },
        variables: params
      }
    }
  })
}
```

4. setup entrypoint loader and container
```tsx
type LoaderData = PreloadedEntryPoint<GetEntryPointComponentFromEntryPoint<typeof PostEntrypoint>>

export const PostEntrypointLoader: LoaderFunction = ({ params }): LoaderData => {
  return loadEntryPoint(
    { getEnvironment: () => environment },
    PostEntrypoint,
    { id: params.id as any }
  )
}

export const PostEntrypointContainer = () => {
  const ref = useLoaderData() as LoaderData

  return (
    <Suspense fallback='Loading post...'>
      <ErrorBoundary fallbackRender={({ error }) => <div>{error.message}</div>}>
        <EntryPointContainer entryPointReference={ref} props={{}} />
      </ErrorBoundary>
    </Suspense>
  )
}
```

5. use with a routing solution (example used with react-router v6 `DataBrowserRouter`)
```tsx
createRoot(document.getElementById('root')!).render(
  <RelayProvider>
    <DataBrowserRouter>
      <Route
        path=':id'
        loader={PostEntrypointLoader}
        element={<PostEntrypointContainer />}
      />
    </DataBrowserRouter>
  </RelayProvider>
)
```