# SDK none breaking change design

## Deprecated APIs

### Deprecate `TeamsFx` class, and update all the template and samples to directly use credentials instead.

### Deprecate `AuthenticationConfiguration` interface and use separate auth config instead

### Deprecate `executionWithToken` and use `executionWithTokenAndConfig` instead

### Deprecate `createMicrosoftGraphClient` and use `createMicrosoftGraphClientWithCredential` instead


### [Optional] Deprecate `TeamsUserCredential`, `OnBehalfOfUserCredential`, `AppCredential`, `TeamsBotSsoPrompt`, `BotSsoConfig`, `BotSsoExecutionDialog`     

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

export type TeamsUserCredentialAuthConfig = {
  initiateLoginEndpoint: string;
  clientId: string;
};

// Current validation logic for clientSecret and  certificateContent is different from below definition
// Current logic will use certificateContent as default, if no certificateContent, then use clientSecret
export type OnBehalfOfCredentialAuthConfig = {
  authorityHost: string;
  clientId: string;
  tenantId: string;
} & (
  | { clientSecret: string; certificateContent?: never }
  | { clientSecret?: never; certificateContent: string }
);

export type AppCredentialAuthConfig = OnBehalfOfCredentialAuthConfig;

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
// Solution 1:
// TeamsUserCredential constructor
constructor(authConfig: TeamsUserCredentialAuthConfig | AuthenticationConfiguration)

// OnBehalfOfUserCredential constructor
constructor(ssoToken: string, config: OnBehalfOfCredentialAuthConfig | AuthenticationConfiguration);

// AppCredential constructor
constructor(authConfig: AppCredentialAuthConfig | AuthenticationConfiguration);


// Solution 2: Change credential name and create a new class:
// TeamsUserCredentialV2 constructor
constructor(authConfig: TeamsUserCredentialAuthConfig)

// OnBehalfOfUserCredentialV2 constructor
constructor(ssoToken: string, config: OnBehalfOfCredentialAuthConfig);

// AppCredentialV2 constructor
constructor(authConfig: AppCredentialAuthConfig);


// Solution3: add a static function to create instance
TeamsUserCredential.create(authConfig: TeamsUserCredentialAuthConfig): TeamsUserCredential

OnBehalfOfUserCredential.create(ssoToken: string, config: OnBehalfOfCredentialAuthConfig):OnBehalfOfUserCredential

// AppCredentialV2 constructor
AppCredential.create(authConfig: AppCredentialAuthConfig):AppCredential
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
// solution 1
constructor(
  private teamsfx: TeamsFx | (OnBehalfOfCredentialAuthConfig & { loginUrl: string })
  dialogId: string,
  private settings: TeamsBotSsoPromptSettings
)

// solution 2
constructor(
  private teamsfx: TeamsFx | OnBehalfOfCredentialAuthConfig
  dialogId: string,
  private settings: TeamsBotSsoPromptSettings,
  loginUrl?: string
)

// solution 3: create a new class use a different name and deprecate TeamsBotSsoPrompt
constructor(
  private authConfig: OnBehalfOfCredentialAuthConfig,
  loginUrl: string,
  dialogId: string,
  private settings: TeamsBotSsoPromptSettings
)

// solution 4: add a static function to create instance
TeamsBotSsoPrompt.create(
  authConfig: OnBehalfOfCredentialAuthConfig,
  loginUrl: string,
  dialogId: string,
  private settings: TeamsBotSsoPromptSettings
): TeamsBotSsoPrompt

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
// Solution 1: 
constructor(
  dedupStorage: Storage,
  ssoPromptSettings: TeamsBotSsoPromptSettings,
  teamsfx: TeamsFx | (OnBehalfOfCredentialAuthConfig & { loginUrl: string })
  dialogName?: string
)

// Solution 2:
constructor(
  dedupStorage: Storage,
  ssoPromptSettings: TeamsBotSsoPromptSettings,
  teamsfx: TeamsFx | OnBehalfOfCredentialAuthConfig
  dialogName?: string,
  loginUrl?: string
)

// Solution 3, create a new class use a different name and deprecate BotSsoExecutionDialog 
constructor(
  dedupStorage: Storage,
  ssoPromptSettings: TeamsBotSsoPromptSettings,
  authConfig: OnBehalfOfCredentialAuthConfig,
  loginUrl: string,
  dialogName?: string
)

// Solution 4: add a static function to create instance
BotSsoExecutionDialog.create(
  dedupStorage: Storage,
  ssoPromptSettings: TeamsBotSsoPromptSettings,
  authConfig: OnBehalfOfCredentialAuthConfig,
  loginUrl: string,
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
);
```

->

```ts
export async function executionWithTokenAndConfig(
  context: TurnContext,
  authConfig: OnBehalfOfCredentialAuthConfig,
  loginUrl: string,
  scopes: string | string[],
  logic?: (token: MessageExtensionTokenResponse) => Promise<any>
);
```

### Update `BotSsoConfig` to `BotSsoConfigV2`:

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
export interface BotSsoConfigV2 {
  aad: {
    scopes: string[];
    loginUrl: string;
  } & OnBehalfOfCredentialAuthConfig;
  ...
}
```

### Update `createMicrosoftGraphClient` to use different credentials:

```ts
export function createMicrosoftGraphClient(
  teamsfx: TeamsFxConfiguration,
  scopes?: string | string[]
): Client;
```

->

```ts
// Change the function name and make original createMicrosoftGraphClient deprecated

// In browser
export function createMicrosoftGraphClientWithCredential(
  credential: TeamsUserCredential,
  scopes?: string | string[]
): Client

// In node
export function createMicrosoftGraphClientWithCredential(
  credential: OnBehalfOfCredential | AppCredential
  scopes?: string | string[]
): Client

```


## API Usage sample:

```ts
// In browser: TeamsUserCredential
const authConfig: TeamsUserCredentialAuthConfig = {
  clientId: "xxx",
  initiateLoginEndpoint: "https://xxx/auth-start.html",
};

const credential = new TeamsUserCredential(authConfig);

const scope = "User.Read";
await credential.login(scope);

const client = createMicrosoftGraphClientWithCredential(credential, scope);
```

```ts
// In node: OnBehalfOfUserCredential
const oboAuthConfig: OnBehalfOfCredentialAuthConfig = {
  authorityHost: "xxx",
  clientId: "xxx",
  tenantId: "xxx",
  clientSecret: "xxx",
};

const oboCredential = new OnBehalfOfUserCredential(ssoToken, oboAuthConfig);
const scope = "User.Read";
const client = createMicrosoftGraphClientWithCredential(oboCredential, scope);
```

```ts
// In node: AppCredential
const appAuthConfig: AppCredentialAuthConfig = {
  clientId: "xxx",
  tenantId: "xxx",
  clientSecret: "xxx",
};
const appCredential = new AppCredential(appAuthConfig);
const scope = "User.Read";
const client = createMicrosoftGraphClientWithCredential(appCredential, scope);
```

## Docs should be updated to help user to choose different credentials for different scenarios:

- Compare different credential, usage scenario.
- Configurations for different scenario.
- Link the credential to official oauth flow introduction document.

## Open questions:
- Rename the class and interface below?
      
      TeamsUserCredential
      OnBehalfOfUserCredential
      AppCredential
      TeamsBotSsoPrompt
      BotSsoConfig
      BotSsoExecutionDialog
      executionWithToken
      createMicrosoftGraphClient



- Naming for the changed API?

      TeamsUserCredential -> TeamsUserCredentialV2
      OnBehalfOfUserCredential -> OnBehalfOfUserCredentialV2
      AppCredential -> AppCredentialV2

      TeamsBotSsoPrompt -> TeamsBotSsoPromptV2
      BotSsoConfig -> BotSsoConfigV2,
      BotSsoExecutionDialog ->BotSsoExecutionDialogV2

      executionWithToken -> executionWithTokenAndConfig
      createMicrosoftGraphClient -> createMicrosoftGraphClientWithCredential

### Already discussed
- `AppCredentialAuthConfig` and `OnBehalfOfCredentialAuthConfig` are the same, do we need to define only one type for these two auth config?

      Use two different types

- Do we need to use another name for the APIs such as below so that to get better code intellisense

  - `createMicrosoftGraphClient` -> `createMicrosoftGraphClientWithCredential`
  - `executionWithToken` -> `executionWithTokenWithConfig`
  - ...

        As discussed, use different names for better intellisense

  

## Issues:

### Already discussed
- Current implementation seems not correct, `loadAndValidateConfig` in `AppCredential` should check whether `authorityHost` exist.

      Will update the SDK to fix this issue: already fixed

- In order not to break current user code, `AuthenticationConfiguration`, `TeamsFx` should be reserved with deprecated notice, and code intellisense may not work.

      Deprecate AuthenticationConfiguration and TeamsFx

- Current validation logic inside `getTediousConnectionConfig` class is different from `SQLAuthConfig`, and current logic will use sqlIdentityId as default, if no sqlIdentityId, then use sqlUserName and password.

        Use the logic in this design
