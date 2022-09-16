---
id: resolver
title: Relay Resolver
sidebar_label: Relay Resolver
---

> Relay Resolvers is an experimental Relay feature which enables modeling derived state as client-only fields in Relay’s GraphQL graph.

> Relay Resolvers were originally conceived of as an alternative to Flux-style selectors and can be thought of as providing similar capabilities.

The under example is from [qr_auth_gql](https://github.com/dohaicuong/qr_auth_gql/tree/master/frontend/src/providers/relay/resolvers)

1. enable relay resolver

`relay.config.json`
```json
{
  "src": ".",
  "schema": "./schema.graphql",
  "excludes": ["**/dist/**", "**/node_modules/**", "**/__generated__/**"],
  "language": "typescript",
  "featureFlags": {
    "enable_relay_resolver_transform": true,
    "enable_client_edges": {
      "kind": "enabled"
    }
  }
}
```

`environment.ts`
```ts
import { RelayFeatureFlags } from 'relay-runtime'
RelayFeatureFlags.ENABLE_RELAY_RESOLVERS = true
```

2. setup script to fix broken artifact
waiting for this [issue](https://github.com/facebook/relay/issues/4067)

`resolver_import_artifact.js`
```js
import fs from 'fs'
import glob from 'glob'

const fixFile = (file) => {
  const data = fs.readFileSync(file)
  let code = data.toString()

  const matchBrokenImports = code.match(/(?<=import\s)(.*)(?=.ts\s)/g)
  if (!matchBrokenImports) return

  const brokenImports = Array.from(matchBrokenImports)
  for (const brokenImport of brokenImports) {
    const regex = new RegExp(`${brokenImport}.ts(?=\\s|,)`, 'g')
    const fix = brokenImport.split('/')[brokenImport.split('/').length - 1]
    code = code.replace(regex, fix)
  }

  fs.writeFileSync(file, code)

  console.log(`fixed ${file}`)
}

glob('src/**/*.graphql.ts', (err, files) => {
  if (err) return console.log(err)

  files.forEach(file => fixFile(file))
})

```

3. setup fix script to run after relay compiler

packages.json
```json
{
  "scripts": {
    "relay": "relay-compiler && pnpm mod:resolvers",
    "mod:resolvers": "node ./scripts/resolver_import_artifact.js"
  }
}
```

4. write resolver

The comment is important, it define the type extention and field name!

`ProfileAgeResolver.ts`
```tsx
import { graphql } from 'relay-runtime'
import { readFragment } from 'relay-runtime/lib/store/ResolverFragments'

/**
 * @RelayResolver
 *
 * @onType Profile
 * @fieldName age
 * @rootFragment ProfileAgeResolver
 *
 * profile user age
 */
export default function ProfileAgeResolver(profileRef: any): number {
  const profile = readFragment(
    graphql`
      fragment ProfileAgeResolver on Profile {
        dob
      }
    `,
    profileRef
  )

  return new Date().getFullYear() - new Date(profile.dob).getFullYear()
}
```

5. use the new field `age`

`ProfileCard/index.tsx`
```tsx
const profile = useFragment(
  graphql`
    fragment ProfileCard_profile on Profile {
      name
      avatar
      title
      description
      age
      ...ProfileEditDialog_profile
    }
  `,
  profileRef
)
```

6. run compiler and fix script

```pnpm relay```