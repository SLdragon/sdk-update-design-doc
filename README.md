# SDK none breaking change design

## Deprecated APIs

- Deprecate `TeamsFx` class, and update all the template and samples to directly use credentials instead.

- Deprecate `AuthenticationConfiguration` interface and use separate auth config instead

- Deprecate `handleMessageExtensionQueryWithToken` and use `handleMessageExtensionQueryWithSSO` instead

- Deprecate `createMicrosoftGraphClient` and use `createMicrosoftGraphClientWithCredential` instead

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
constructor(authConfig: TeamsUserCredentialAuthConfig);
constructor(authConfig: AuthenticationConfiguration);
constructor(authConfig: TeamsUserCredentialAuthConfig | AuthenticationConfiguration)

// OnBehalfOfUserCredential constructor
constructor(ssoToken: string, config: OnBehalfOfCredentialAuthConfig);
constructor(ssoToken: string, config: AuthenticationConfiguration);
constructor(ssoToken: string, config: OnBehalfOfCredentialAuthConfig | AuthenticationConfiguration);


// AppCredential constructor
constructor(authConfig: AppCredentialAuthConfig);
constructor(authConfig: AuthenticationConfiguration);
constructor(authConfig: AppCredentialAuthConfig | AuthenticationConfiguration)

/* Other abandoned solution

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
*/
~~~
```

### Update `TeamsBotSsoPrompt` to use `OnBehalfOfCredentialAuthConfig` and `initiateLoginEndpoint`:

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
  constructor(teamsfx: TeamsFx, dialogId: string, settings: TeamsBotSsoPromptSettings);
  constructor(
    authConfig: OnBehalfOfCredentialAuthConfig,
    initiateLoginEndpoint: string,
    dialogId: string,
    settings: TeamsBotSsoPromptSettings
  );
  constructor() 

/* Other abandoned solution
// solution 1
constructor(
  private teamsfx: TeamsFx | (OnBehalfOfCredentialAuthConfig & { loginUrl: string })
  dialogId: string,
  private settings: TeamsBotSsoPromptSettings
)
// solution 2
constructor(jgdsrt
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
*/

```

### Update `BotSsoExecutionDialog`

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
    teamsfx: TeamsFx,
    dialogName?: string
  );
  constructor(
    dedupStorage: Storage,
    ssoPromptSettings: TeamsBotSsoPromptSettings,
    authConfig: OnBehalfOfCredential,
    initiateLoginEndpoint: string,
    dialogName?: string
  );
  constructor()


/* Other abandoned solution
// Solution 1: 
constructor(
  dedupStorage: Storage,
  ssoPromptSettings: TeamsBotSsoPromptSettings,
  teamsfx: TeamsFx | (OnBehalfOfCredentialAuthConfig & { loginUrl: string })
  dialogName?: string
)

// Solution 3, create a new class use a different name and deprecate BotSsoExecutionDialog 
constructor(
  dedupStorage: Storage,
  ssoPromptSettings: TeamsBotSsoPromptSettings,
  authConfig: OnBehalfOfCredentialAuthConfig,
  loginUrl: string,
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

// Solution 4: add a static function to create instance
BotSsoExecutionDialog.create(
  dedupStorage: Storage,
  ssoPromptSettings: TeamsBotSsoPromptSettings,
  authConfig: OnBehalfOfCredentialAuthConfig,
  loginUrl: string,
  dialogName?: string
)
*/

```

### Update message extension `executionWithToken` function to use `OnBehalfOfCredential` and `initiateLoginEndpoint`:

```ts
// original
export async function handleMessageExtensionQueryWithToken(
  context: TurnContext,
  config: AuthenticationConfiguration | null,
  scopes: string | string[],
  logic: (token: MessageExtensionTokenResponse) => Promise<any>
)
```

->

```ts
export async function handleMessageExtensionQueryWithSSO(
  context: TurnContext,
  config: OnBehalfOfCredential | null,
  initiateLoginEndpoint: string,
  scopes: string | string[],
  logic: (token: MessageExtensionTokenResponse) => Promise<any>
)
```

### Update `BotSsoConfig` to use `OnBehalfOfCredentialAuthConfig` and `initiateLoginEndpoint`:

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
  }
  | (OnBehalfOfCredentialAuthConfig & { initiateLoginEndpoint: string })
  | AuthenticationConfiguration;
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

export function createMicrosoftGraphClientWithCredential(
  credential: TokenCredential,
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

  Only rename the below class and interface:

        handleMessageExtensionQueryWithToken
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

  Rename as below:

      handleMessageExtensionQueryWithToken -> handleMessageExtensionQueryWithSSO
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
