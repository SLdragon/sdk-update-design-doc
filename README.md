# SDK none breaking change design

## Deprecated APIs

### Deprecate `TeamsFx` class, and update all the template and samples to directly use credentials instead.
### Deprecate `AuthenticationConfiguration` interface

## Code Update

### Add different auth configurations to help user to know exactly what properties needed for different scenarios

```ts
// original auth config, add deprecate notice
export interface AuthenticationConfiguration {
  readonly authorityHost?: string;
  readonly tenantId?: string;
  readonly clientId?: string;
  readonly clientSecret?: string;
  readonly certificateContent?: string;
  readonly initiateLoginEndpoint?: string;
  readonly applicationIdUri?: string;
}
```
->

```ts
export type OnBehalfOfCredentialAuthConfig = {
  authorityHost: string;
  clientId: string;
  tenantId: string;
} & (
  | { clientSecret: string; certificateContent?: never }
  | { clientSecret?: never; certificateContent: string }
);

export type AppCredentialAuthConfig = OnBehalfOfCredentialAuthConfig;

export type TeamsUserCredentialAuthConfig = {
  initiateLoginEndpoint: string;
  clientId: string;
};

export type MsgExtAuthConfig = {
  authorityHost: string;
  clientId: string;
  tenantId: string;
  initiateLoginEndpoint: string;
} & (
  | { clientSecret: string; certificateContent?: never }
  | { clientSecret?: never; certificateContent: string }
);

export type BotSsoAuthConfig = {
  authorityHost: string;
  clientId: string;
  tenantId: string;
  initiateLoginEndpoint: string;
  applicationIdUri: string;
} & (
  | { clientSecret: string; certificateContent?: never }
  | { clientSecret?: never; certificateContent: string }
)

// Current validation logic inside getTediousConnectionConfig class is different from below definition
// Current logic will use sqlIdentityId as default, if no sqlIdentityId, then use sqlUserName and password
export type SQLAuthConfig = {
  sqlServerEndpoint: string;
} & (
  | { sqlUsername: string; sqlPassword: string; sqlIdentityId?: never }
  | { sqlUsername?: never; sqlPassword?: never; sqlIdentityId: string }
);
```

### Update credential constructor to use different auth config:
```ts
// TeamsUserCredential constructor
constructor(authConfig: AuthenticationConfiguration);

// OnBehalfOfUserCredential constructor
constructor(ssoToken: string, config: AuthenticationConfiguration);

// AppCredential constructor
constructor(authConfig: AuthenticationConfiguration);
```

->

```ts
// TeamsUserCredential constructor 
constructor(authConfig: TeamsUserCredentialAuthConfig | AuthenticationConfiguration) 

// OnBehalfOfUserCredential constructor
constructor(ssoToken: string, config: OnBehalfOfCredentialAuthConfig | AuthenticationConfiguration);

// AppCredential constructor
constructor(authConfig: AppCredentialAuthConfig | AuthenticationConfiguration);
```


### Update `TeamsBotSsoPrompt` to use `BotSsoAuthConfig`:
```ts
// original
constructor(
  private teamsfx: TeamsFx,
  dialogId: string,
  private settings: TeamsBotSsoPromptSettings
)
```

->

```ts
constructor(
  private teamsfx: TeamsFx | BotSsoAuthConfig,
  dialogId: string,
  private settings: TeamsBotSsoPromptSettings
)
``` 

### Update `BotSsoExecutionDialog` to use `BotSsoAuthConfig`
```ts
constructor(
  dedupStorage: Storage,
  ssoPromptSettings: TeamsBotSsoPromptSettings,
  teamsfx: TeamsFx,
  dialogName?: string
)
```
->
```ts
constructor(
  dedupStorage: Storage,
  ssoPromptSettings: TeamsBotSsoPromptSettings,
  teamsfx: TeamsFx | BotSsoAuthConfig,
  dialogName?: string
)
```


### Update message extension `executionWithToken` function to use `MsgExtAuthConfig`:
```ts
// original
export async function executionWithToken(
  context: TurnContext,
  config: AuthenticationConfiguration,
  scopes: string | string[],
  logic?: (token: MessageExtensionTokenResponse) => Promise<any>
)
```
->
```ts
export async function executionWithToken(
  context: TurnContext,
  config: AuthenticationConfiguration | MsgExtAuthConfig,
  scopes: string | string[],
  logic?: (token: MessageExtensionTokenResponse) => Promise<any>
)
```


### Update `BotSsoConfig` to use  `BotSsoAuthConfig`:
```ts
export interface BotSsoConfig {
  aad: {
    scopes: string[];
  } & AuthenticationConfiguration;
  ...
}
```
->
```ts
export interface BotSsoConfig {
  aad: {
    scopes: string[];
  } & (AuthenticationConfiguration | BotSsoAuthConfig);
  ...
}
```

### Update `createMicrosoftGraphClient` to use different credentials:
```ts
export function createMicrosoftGraphClient(
  teamsfx: TeamsFxConfiguration,
  scopes?: string | string[]
): Client
```

->

```ts
// In browser
export function createMicrosoftGraphClient(
  teamsfx: TeamsFxConfiguration | TeamsUserCredential,
  scopes?: string | string[]
): Client

// In node
export function createMicrosoftGraphClient(
  teamsfx: TeamsFxConfiguration | OnBehalfOfCredential | AppCredential
  scopes?: string | string[]
): Client

```

### Update `createConfidentialClientApplication` signature in `utils.node.ts`:
```ts
export function createConfidentialClientApplication(
  authentication: AuthenticationConfiguration
): ConfidentialClientApplication
```
->
```ts
export function createConfidentialClientApplication(
  authentication:
    | AuthenticationConfiguration
    | OnBehalfOfCredentialAuthConfig
    | AppCredentialAuthConfig
): ConfidentialClientApplication

```

## API Usage sample:
```ts
// In browser: TeamsUserCredential
const authConfig: TeamsUserCredentialAuthConfig = {
  clientId: "xxx",
  initiateLoginEndpoint: "https://xxx/auth-start.html"
}

const credential = new TeamsUserCredential(authConfig);

const scope= "User.Read";
await credential.login(scope);

const client = createMicrosoftGraphClient(credential, scope);
```

```ts
// In node: OnBehalfOfUserCredential
const oboAuthConfig: OnBehalfOfCredentialAuthConfig = {
  authorityHost: "xxx",
  clientId: "xxx",
  tenantId: "xxx",
  clientSecret: "xxx"
}

const oboCredential = new OnBehalfOfUserCredential(ssoToken, oboAuthConfig);
const scope = "User.Read";
const client = createMicrosoftGraphClient(oboCredential, scope);
```

```ts
// In node: AppCredential
const appAuthConfig: AppCredentialAuthConfig = {
  clientId: "xxx",
  tenantId: "xxx",
  clientSecret: "xxx"
}
const appCredential = new AppCredential(appAuthConfig);
const scope = "User.Read";
const client = createMicrosoftGraphClient(appCredential, scope);
```

## Docs should be updated to help user to choose different credentials for different scenarios:

- Compare different credential, usage scenario.
- Configurations for different scenario.
- Link the credential to official oauth flow introduction document.


## Open questions:
- `AppCredentialAuthConfig` and `OnBehalfOfCredentialAuthConfig` are the same, do we need to define only one type for these two auth config?
- Do we need to use another name for the APIs such as below so that to get better code intellisense
  - `createMicrosoftGraphClient` -> `createMicrosoftGraphClientWithCredential`
  - `executionWithToken` -> `executionWithTokenV2`
  - ...


## Issues:
- Current implementation seems not correct, `loadAndValidateConfig` in `AppCredential` should check whether `authorityHost` exist.
- In order not to break current user code, `AuthenticationConfiguration`, `TeamsFx` should be reserved with deprecated notice, and code intellisense may not work.
- Current validation logic inside `getTediousConnectionConfig` class is different from `SQLAuthConfig`, and current logic will use sqlIdentityId as default, if no sqlIdentityId, then use sqlUserName and password
