---
title: "[译]Cookies vs Tokens : 权威指南"
date: 2018-05-10 10:59:24
update: 2018-05-10 14:59:24
categories:
    - 接口测试
tags:
    - cookies
    - tokens
    - JWT
---

本文译自：[Cookies vs Tokens: The Definitive Guide](https://auth0.com/blog/cookies-vs-tokens-definitive-guide/)

********

# TL;DR

基于令牌的身份验证（Tokens-based authentication，下文简写为token）正在变得越来越流行。 我们将研究cookie和token之间的区别和相似之处，使用token的优势，以及解决开发人员关于token的常见问题和疑虑。 最后，我们将构建一个使用token并使其成为渐进式Web应用程序（PWA）的应用程序。

我们将编写一个使用JWT进行身份验证的Angular 2应用程序。源码在此: [GitHub repo](https://github.com/auth0-blog/angular-auth0-aside)。

<!-- more -->
********

我们[上一篇](https://auth0.com/blog/2014/01/07/angularjs-authentication-with-cookies-vs-token/)对比cookie和token的文章是在两年前*(译者注：本文发表于2016年5月31日)*。从那以后，我们已经写了很多关于如何在不同语言和框架中集成token的文章。

单页面应用程序（SPA）的兴起以及前后端分离是非常有效的。 像Angular、React和Vue这样的框架允许开发人员构建比以前更强大、更高性能的单页应用程序。 而token可以与这些框架结合在一起使用。

>"Token-based authentication goes hand in hand with SPA frameworks like Angular, React and Vue."

# Cookie vs Token Authentication - 概览

在我们进一步深入之前，让我们快速回顾一下这两种认证系统是如何工作的。如果您已经熟悉cookie和token的工作原理，请跳过本节，否则请继续阅读。

下图简要概述了cookie和token认证方法之间的区别。
{% asset_img cookie-token-auth.png [cookie-token-auth] %}

## 基于Cookie的身份验证

基于Cookie的身份验证早已成为处理用户认证的默认和可靠的方法。

基于cookie的身份验证是**有状态的**。这意味着认证记录或会话（session）在服务器端和客户端都要保存。服务器需要跟踪数据库中的活动session，而在前端也要创建一个包含session标识符的cookie，从而实现基于cookie的身份验证。我们来看看传统的基于cookie的认证流程：

1. 用户输入他们的登录凭证
2. 服务器验证凭据是否正确，并创建一个session，然后将其存储在数据库中
3. 具有session ID的cookie被放置在用户浏览器中（译者注：注意这里是session ID，不是session）
4. 在随后的请求中，将根据数据库验证session ID，并且判断请求是否有效
5. 一旦用户注销了应用程序，客户端和服务器端的session就会被销毁

## 基于Token的身份验证

由于SPA、Web API和物联网（IoT）的兴起，token在过去几年得到了普及。当我们谈论token时，我们通常会谈论使用[JSON Web Tokens](https://jwt.io/introduction)（JWT）进行身份验证。尽管实现token的方式有所不同，但JWT已成为事实上的标准。考虑到这种情况，本文的其余部分将交替使用token和JWT。

基于token的认证是**无状态的**。服务器不记录哪些用户已登录或哪些JWT已发出。与之相对，对服务器的每个请求都伴随着一个token，服务器使用该token来验证请求的真实性。该token通常以{JWT}的形式作为附加header发送，但也可以在POST请求的主体中发送，甚至作为查询参数发送。让我们看看这个流程是如何工作的：

1. 用户输入他们的登录凭证
2. 服务器验证凭据是否正确并返回签名过的token
3. 该token存储在客户端，通常存储在本地存储（local storage）中，但也可以存储在会话存储（session storage）或cookie中
4. 对服务器的后续请求将此token包含为额外的header，或通过上述其他方法之一
5. 服务器解码JWT，如果token有效，则处理该请求
6. 一旦用户注销，token就会被销毁，并且客户端不需要与服务器交互

# 基于Token身份验证的优势

只了解认证过程还不够。接下来，我们将介绍token优于cookie方案的原因。

## 无状态，可扩展和解耦

将token存储在cookie中，最大的好处可能在于token是无状态的。后端不需要记录token。每个token都是独立的，包含检查其有效性以及通过声明传达用户信息所需的所有数据。

服务器的唯一工作就是在成功的登录请求上签名token，并验证传入token是否有效。~~事实上，服务器甚至不需要对token进行签名。第三方服务（如Auth0）可以处理token的发放，然后服务器只需验证token的有效性。~~（译者注：作者夹带私货）

## 跨域（Cross Domain）和跨域资源共享（CORS）

Cookie可以很好地处理单个域名和子域名，但是当涉及跨不同域名管理Cookie时，它可能会变得复杂。相反，token使得将API公开给不同的服务和域变得简单。每次调用后端时，都通过JWT对token进行校验，只要token有效就可以处理请求。这里有一些注意事项，我们将在下面的常见问题和关注部分讨论这些问题。

## 将数据存储在JWT中

在使用基于cookie的身份验证时，您只是将session ID存储在cookie中。而JWT则允许您存储任何类型的元数据，只要它是有效的JSON。[JWT规范](https://tools.ietf.org/html/rfc7519)指定了元数据中可以包含的不同类型的声明，例如保留声明，公共声明和私有声明。您可以通过[jwt.io网站](https://jwt.io/introduction/)了解各种声明的具体情况和差异。

实际上，这意味着JWT可以包含任何类型的数据。根据您的使用情况，您可以选择提供最少的声明，例如用户ID和token到期时间；或者您可以决定添加其他声明，例如用户电子邮件地址，颁发token的人员，范围或用户权限等等。

## 性能

在使用基于cookie的身份验证时，后端必须执行查询，无论是传统SQL数据库还是NoSQL，并且与解码token相比，整个流程可能需要更长的时间。此外，由于您可以将其他数据存储在JWT中，例如用户权限级别，因此您可以省去额外的获取和处理请求的数据的调用。
例如，假设您有一个，用于检索最新订单的API`/api/orders`，但只有admin权限的角色才能查看此数据。在基于cookie的方法中，一旦发出请求，您将需要一次请求数据库来验证会话有效，另一次请求以获取用户数据并验证用户是否具有admin权限，最后是第三个请求来获取数据。使用JWT方法，您可以将用户角色存储在JWT中，因此一旦提出请求并验证了JWT，您就可以对数据库进行唯一一次请求以检索订单。

## 移动端支持

现代API不仅与浏览器交互。正确编写一个API也可以为iOS和Android等浏览器和Native APP提供服务。原生移动平台和Cookie不能很好地融合。虽然可能，但在移动平台上使用cookie有许多限制和考虑因素。另一方面，token在iOS和Android上都更容易实现。对于没有cookie存储概念的物联网应用程序和服务，令牌也更容易实现。

# 常见问题和疑虑

在本节中，我们将看看token认证时经常出现的一些常见问题和疑虑。这里关注的焦点将是安全性，同时我们也会关注token的大小、存储和加密。

## JWT的大小

token认证的最大缺点是JWT的大小。session cookie甚至比最小的JWT还要小。根据您的使用情况，如果您添加了多个声明，token的大小可能会变成一个问题。请记住，对服务器的每个请求都必须包含JWT。

## 在哪里存储token？

使用基于token的身份验证，您可以选择在哪里存储JWT。通常，JWT放置在浏览器本地存储（local storage）中，对于大多数使用情况来说，这种方式非常有效。将JWT存储在local storage中时需要注意一些问题。与cookie不同的是，local storage被沙箱化为特定的域，其数据不能被任何其他域（包括子域）访问。
您可以将token存储在cookie中，但cookie的最大大小仅为4kb，因此如果您有多个声明，则可能会产生问题。此外，您可以将token存储在与local storage类似的session storage中，但只要用户关闭浏览器就会被清除。

## XSS和XSRF防护

保护用户和服务器始终是首要任务。开发人员在决定是否使用基于token的身份验证时最常遇到的问题之一是安全隐患。面向网站的两种最常见的攻击媒介是跨站脚本（XSS）和跨站请求伪造（XSRF或CSRF）。
[XSS](https://zh.wikipedia.org/wiki/%E8%B7%A8%E7%B6%B2%E7%AB%99%E6%8C%87%E4%BB%A4%E7%A2%BC) - 当外部实体能够在您的网站或应用程序内执行代码时发生攻击。这里最常见的攻击方法是，如果你的网站允许输入的信息没有被正确处理。如果攻击者可以在您的域上执行代码，那么您的JWT token易受攻击。我们的首席技术官[过去认为](https://auth0.com/blog/ten-things-you-should-know-about-tokens-and-cookies/#xss-xsrf)，与XSRF攻击相比，XSS攻击更容易防范。许多框架，包括Angular，都会自动清理输入并防止任意代码执行。如果你没有使用框架来清理输入/输出，你可以尝试像google开发的[caja插件](https://github.com/google/caja)。过滤输入是许多框架和语言中尝试解决的问题，我会推荐使用框架或插件来构建自己的输入。
如果您在local storage中使用JWT，则跨站请求伪造攻击不是问题。另一方面，如果您期望JWT存储在cookie中，则需要防范XSRF。XSRF的原理不像XSS攻击那么容易理解。解释XSRF攻击如何工作可能非常耗时，所以，请查看[本指南](https://en.wikipedia.org/wiki/Cross-site_request_forgery)，深入解释XSRF攻击如何工作。幸运的是，防范XSRF攻击是一件相当简单的事情。为了防止XSRF攻击，您的服务器在建立与客户端的会话时将生成唯一的token（请注意，这不是JWT）。然后，无论何时将数据提交给服务器，隐藏的输入字段都将包含此token，并且服务器将检查以确保token匹配。同样，由于我们的建议是将JWT存储在local storage中，因此您可能不必担心XSRF攻击。
保护用户和服务器的最佳方式之一是token的过期时间很短。这样，即使token被泄露，它也会很快变得无效。此外，您可能会维护一个被盗用token的[黑名单](https://auth0.com/blog/2015/03/10/blacklist-json-web-token-api-keys/)，并且不允许这些令牌访问系统。您还可以通过改变签名算法作为最终手段，这会使所有有效令牌失效并要求所有用户再次登录。当然这种方法不推荐，但在严重情况下可用。

## token已签名，未加密

一个JWT由三部分组成：header, payload和signature。JWT的格式是`header.payload.signature`。如果我们要用HMACSHA256算法签名JWT，secret是'shhhh'和下面的payload：

```json
{
  "sub": "1234567890",
  "name": "Ado Kukic",
  "admin": true
}
```

生成的JWT将是：

    QiOjE0NjQyOTc4ODV9.Y47kJvnHzU9qeJIN48_bVna6O0EDFiMiQ9LpNVDFymM

这里需要**注意**的是，这个token是由HMACSHA256算法签名的，header和payload是Base64URL编码的，它**没有加密**。如果我访问[jwt.io](https://jwt.io/)，粘贴此token并选择HMACSHA256算法，我可以解码并读取其内容。因此，不应该将敏感数据（如密码）存储在payload中。
如果您必须将敏感数据存储在有效内容中，或者您需要屏蔽JWT，则可以使用JSON Web Encryption（JWE）。JWE允许您加密JWT的内容，以便除服务器之外的任何人都无法读取它。[JOSE](http://jose.readthedocs.io/en/latest/)为JWE提供了一个很好的框架和不同的选项，并且为许多流行的框架（包括[NodeJS](https://github.com/cisco/node-jose)和[Java](https://bitbucket.org/connect2id/nimbus-jose-jwt/wiki/Home)）提供了SDK。无论如何，我鼓励您了解更多关于[AngularJS身份验证](https://auth0.com/learn/angularjs-authentication/)的信息。

# 使用Auth0进行基于Token的身份验证

*以下为机翻内容*
在Auth0中，我们编写了SDK，指南和快速入门，用于与JWT一起使用多种语言和框架，包括[NodeJS](https://github.com/auth0/express-jwt)，[Java](https://github.com/auth0/java-jwt)，[Python](https://github.com/auth0/auth0-python)，[GoLang](https://github.com/auth0/go-jwt-middleware), [等等](https://auth0.com/docs)。我们最后的“Cookies与Tokens”文章使用了AngularJS框架，所以今天我们的代码样本适合使用Angular 2。
您可以从我们的由[Kim Maida](https://twitter.com/kimmaida)创建的[GitHub仓库](https://github.com/auth0-blog/angular-auth0-aside)中下载示例代码。下载代码样本比较好，因为Angular 2需要很多初始设置才能开始。如果您尚未注册，请[注册](https://auth0.com/signup)一个免费的Auth0帐户，以便您可以自己执行实施并尝试不同的功能和选项。让我们开始吧

# 另外：使用Auth0验证Angular App和Node API

我们可以保护我们的应用程序和API，以便只有经过身份验证的用户才能访问它们。我们来看看如何使用[Auth0](https://auth0.com/)来实现Angular应用程序和Node API。您可以从[GitHub](https://github.com/auth0-blog/angular-auth0-aside)上的angular-auth0-aside回购中克隆此示例应用程序和API。
{% asset_img auth0-centralized-login.jpg [auth0-centralized-login] %}

## 功能

示例Angular应用程序和API具有以下功能：

- Angular CLI生成Angular应用程序，并在`http://localhost:4200`处提供服务
- 使用登录页面进行[auth0.js](https://auth0.com/docs/libraries/auth0js/v8)身份验证
- 节点服务器保护的API路由`http://localhost:3001/api/dragons`返回经过验证的`GET`请求的JSON数据
- 一旦用户使用Auth0进行身份验证，Angular app就会从API中获取数据
- 配置文件页面需要使用路由防护进行身份验证
- 身份验证服务使用主题将身份验证状态事件传播到整个应用程序
- 用户配置文件在认证时被提取并存储在认证服务中
- 访问令牌，配置文件和令牌到期存储在本地存储中，并在注销时删除

## 注册Auth0

您需要一个[Auth0](https://auth0.com/)帐户来管理身份验证。您可以在[这里](https://auth0.com/signup)注册一个免费帐户。接下来，设置Auth0应用程序和API，以便Auth0可以与Angular应用程序和Node API进行交互。

## 设置一个Auth0应用程序

1. 转到您的Auth0仪表板并点击“创建新应用程序”按钮。
2. 命名您的新应用程序并选择“单页网络应用程序”。
3. 在新的Auth0应用的设置中，将`http://localhost:4200/callback`添加到**允许的回调URL**中。点击“保存更改”按钮。
4. 如果你愿意，你可以[建立一些社交关系](https://manage.auth0.com/#/connections/social)。然后，您可以在**连接**选项卡下的**应用程序**选项中为您的应用程序启用它们。上面的屏幕截图中显示的示例使用用户名/密码数据库，Facebook，Google和Twitter。对于制作，请确保您设置了自己的社交密钥，并且不要将社交连接设置为使用Auth0开发密钥。

> **注意**：在**高级设置**的**OAuth**选项卡下（**设置**部分的底部），您应该看到**JsonWebToken签名算法**设置为`RS256`。这是新应用程序的默认设置。如果设置为`HS256`，请将其更改为`RS256`。您可以在[这里](https://community.auth0.com/questions/6942/jwt-signing-algorithms-rs256-vs-hs256)阅读有关RS256与HS256 JWT签名算法的更多信息。

## 设置一个API

1. 转到Auth0仪表板中的**[API](https://manage.auth0.com/#/apis)**，然后单击“创建API”按钮。输入API的名称。将**标识符**设置为您的API端点URL。在这个例子中，这是`http://localhost:3001/api/`。签名算法应该是`RS256`。
2. 您可以参阅新API的设置中的**快速启动**选项卡下的Node.js示例。我们将以这种方式实现我们的Node API，使用[Express](https://expressjs.com/)，[express-jwt](https://github.com/auth0/express-jwt)和[jwks-rsa](https://github.com/auth0/node-jwks-rsa)。

我们现在准备在我们的Angular客户端和Node后端API上实现Auth0认证。

## 依赖和设置

Angular应用程序使用[Angular CLI](https://github.com/angular/angular-cli)。确保您已全局安装CLI：

```bash
npm install -g @angular/cli
```

在克隆[项目](https://github.com/auth0-blog/angular-auth0-aside)后，通过在项目文件夹的根目录中运行以下命令来安装Angular应用程序和Node服务器的节点依赖项：

```bash
npm install
cd server
npm install
```

Node API位于示例应用程序根目录下的[`/server`文件夹](https://github.com/auth0-blog/angular-auth0-aside/tree/master/server)中。
找到[`config.js.example`文件](https://github.com/auth0-blog/angular-auth0-aside/blob/master/server/config.js.example)并从文件名中删除`.example`扩展名。然后打开文件：

```js
// server/config.js (formerly config.js.example)
module.exports = {
  CLIENT_DOMAIN: '[AUTH0_CLIENT_DOMAIN]', // e.g. 'you.auth0.com'
  AUTH0_AUDIENCE: 'http://localhost:3001/api/'
};
```

将`AUTH0_CLIENT_DOMAIN`标识符更改为您的Auth0应用程序域，并将`AUTH0_AUDIENCE`设置为您的受众（在本例中为`http://localhost:3001/api/`）。 `/api/dragons`路由将受到[express-jwt](https://github.com/auth0/express-jwt)和[jwks-rsa](https://github.com/auth0/node-jwks-rsa)的保护。

> **注意**：要了解有关RS256和JSON Web密钥集的更多信息，请阅读[导航RS256和JWKS](https://auth0.com/blog/navigating-rs256-and-jwks/)。

我们的API现在受到保护，所以让我们确保我们的Angular应用程序也可以与Auth0进行交互。为此，我们将通过从文件扩展名中删除`.example`来激活[`src/app/auth/auth0-variables.ts.example`文件](https://github.com/auth0-blog/angular-auth0-aside/blob/master/src/app/auth/auth0-variables.ts.example)。然后打开文件并将`[AUTH0_CLIENT_ID]`和`[AUTH0_CLIENT_DOMAIN]`字符串更改为Auth0信息：

```js
// src/app/auth/auth0-variables.ts (formerly auth0-variables.ts.example)
...
export const AUTH_CONFIG: AuthConfig = {
  CLIENT_ID: '[AUTH0_CLIENT_ID]',
  CLIENT_DOMAIN: '[AUTH0_CLIENT_DOMAIN]',
  ...
```

我们的应用和API现在已经建立。它们可以通过运行根文件夹中的`ng serve`和来自`/server`文件夹的`node server.js`来运行。
随着Node API和Angular应用程序的运行，让我们来看看如何实现身份验证。

## 认证服务

使用`AuthService`认证服务处理前端的认证逻辑：[`src/app/auth/auth.service.ts`文件](https://github.com/auth0-blog/angular-auth0-aside/blob/master/src/app/auth/auth.service.ts)。

```js
// src/app/auth/auth.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';
import * as auth0 from 'auth0-js';
import { AUTH_CONFIG } from './auth0-variables';
import { UserProfile } from './profile.model';

(window as any).global = window;

@Injectable()
export class AuthService {
  // Create Auth0 web auth instance
  // @TODO: Update AUTH_CONFIG and remove .example extension in src/app/auth/auth0-variables.ts.example
  private _auth0 = new auth0.WebAuth({
    clientID: AUTH_CONFIG.CLIENT_ID,
    domain: AUTH_CONFIG.CLIENT_DOMAIN,
    responseType: 'token',
    redirectUri: AUTH_CONFIG.REDIRECT,
    audience: AUTH_CONFIG.AUDIENCE,
    scope: AUTH_CONFIG.SCOPE
  });
  userProfile: UserProfile;
  accessToken: string;

  // Create a stream of logged in status to communicate throughout app
  loggedIn: boolean;
  loggedIn$ = new BehaviorSubject<boolean>(this.loggedIn);

  constructor() {
    // You can restore an unexpired authentication session on init
    // by using the checkSession() endpoint from auth0.js:
    // https://auth0.com/docs/libraries/auth0js/v9#using-checksession-to-acquire-new-tokens
  }

  private _setLoggedIn(value: boolean) {
    // Update login status subject
    this.loggedIn$.next(value);
    this.loggedIn = value;
  }

  login() {
    // Auth0 authorize request
    this._auth0.authorize();
  }

  handleLoginCallback() {
    // When Auth0 hash parsed, get profile
    this._auth0.parseHash((err, authResult) => {
      if (authResult && authResult.accessToken) {
        window.location.hash = '';
        this.getUserInfo(authResult);
      } else if (err) {
        console.error(`Error: ${err.error}`);
      }
    });
  }

  getUserInfo(authResult) {
    // Use access token to retrieve user's profile and set session
    this._auth0.client.userInfo(authResult.accessToken, (err, profile) => {
      this._setSession(authResult, profile);
    });
  }

  private _setSession(authResult, profile) {
    const expTime = authResult.expiresIn * 1000 + Date.now();
    // Save session data and update login status subject
    localStorage.setItem('expires_at', JSON.stringify(expTime));
    this.accessToken = authResult.accessToken;
    this.userProfile = profile;
    this._setLoggedIn(true);
  }

  logout() {
    // Remove token and profile and update login status subject
    localStorage.removeItem('expires_at');
    this.accessToken = undefined;
    this.userProfile = undefined;
    this._setLoggedIn(false);
  }

  get authenticated(): boolean {
    // Check if current date is greater than expiration
    // and user is currently logged in
    const expiresAt = JSON.parse(localStorage.getItem('expires_at'));
    return (Date.now() < expiresAt) && this.loggedIn;
  }

}
```

该服务使用来自`auth0-variables.ts`的配置变量实例化`auth0.js`WebAuth实例。
[RxJS `BehaviorSubject`](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/subjects/behaviorsubject.md)用于提供一系列身份验证状态事件，您可以在应用程序的任何位置订阅这些事件。
`login()`方法使用你的配置变量授权带有Auth0的认证请求。登录页面将显示给用户，然后他们可以登录。

> **注意**：如果用户第一次访问我们的应用程序，并且我们的回调位于`localhost`，那么他们还会看到一个同意屏幕，他们可以授予对我们API的访问权限。非本地主机域上的第一方客户端将被高度信任，因此在这种情况下不会出现同意对话框。您可以通过编辑[Auth0 Dashboard API](https://manage.auth0.com/#/apis)**设置**来修改此内容。查找“允许跳过用户同意”切换。

返回到我们的应用程序时，我们将在Auth0的哈希中接收`accessToken`和`expiresIn`。 `handleLoginCallback()`方法使用Auth0的`parseHash()`方法回调来获取用户的配置文件（`getUserInfo()`），并通过保存令牌，配置文件和令牌过期并更新`loggedIn$`主题来设置会话（`_setSession()`），通知应用程序中订阅的组件已通过身份验证。

> **注意**：该配置文件采用[OpenID标准声明](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims)中[`profile.model.ts`](https://github.com/auth0-blog/angular-auth0-aside/blob/master/src/app/auth/profile.model.ts)的形式。

最后，我们有一个`logout()`方法，用于清除数据并更新`loggedIn$`主题。我们也有一个`authenticated`的访问者，根据令牌的存在和令牌的有效期限返回当前的身份验证状态。
一旦在[`app.module.ts`中提供了`AuthService`](https://github.com/auth0-blog/angular-auth0-aside/blob/master/src/app/app.module.ts#L32)，其方法和属性就可以在我们的应用程序的任何地方使用，例如[家庭组件](https://github.com/auth0-blog/angular-auth0-aside/tree/master/src/app/home)。

## 回调组件

[回调组件](https://github.com/auth0-blog/angular-auth0-aside/tree/master/src/app/callback)是认证后应用程序重定向的地方。该组件只是在登录过程完成之前显示加载消息。它执行`handleLoginCallback()`方法来解析散列并提取认证信息。它在我们的认证服务中订阅了`loggedIn$` Behaviour Subject，以便在用户登录后重定向回主页，如下所示：

```js
// src/app/callback/callback.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subscription } from 'rxjs';
import { AuthService } from './../auth/auth.service';
import { Router } from '@angular/router';

@Component({
  selector: 'app-callback',
  templateUrl: './callback.component.html',
  styleUrls: ['./callback.component.css']
})
export class CallbackComponent implements OnInit, OnDestroy {
  loggedInSub: Subscription;

  constructor(private auth: AuthService, private router: Router) {
    // Parse authentication hash
    auth.handleLoginCallback();
  }

  ngOnInit() {
    this.loggedInSub = this.auth.loggedIn$.subscribe(
      loggedIn => loggedIn ? this.router.navigate(['/']) : null
    )
  }

  ngOnDestroy() {
    this.loggedInSub.unsubscribe();
  }

}
```

## 进行经过身份验证的API请求

为了进行经过验证的HTTP请求，我们需要在我们的[`api.service.ts`文件](https://github.com/auth0-blog/angular-auth0-aside/blob/master/src/app/api.service.ts)中添加带有`Authorization`的标头。

```js
// src/app/api.service.ts
import { Injectable } from '@angular/core';
import { throwError, Observable } from 'rxjs';
import { HttpClient, HttpHeaders, HttpErrorResponse } from '@angular/common/http';
import { catchError } from 'rxjs/operators';
import { AuthService } from './auth/auth.service';

@Injectable()
export class ApiService {
  private baseUrl = 'http://localhost:3001/api/';

  constructor(
    private http: HttpClient,
    private auth: AuthService
  ) { }

  getDragons$(): Observable<any[]> {
    return this.http
      .get<any[]>(`${this.baseUrl}dragons`, {
        headers: new HttpHeaders().set(
          'Authorization', `Bearer ${this.auth.accessToken}`
        )
      })
      .pipe(
        catchError(this._handleError)
      );
  }

  private _handleError(err: HttpErrorResponse | any) {
    const errorMsg = err.message || 'Unable to retrieve data';
    return throwError(errorMsg);
  }

}
```

## 最后：路由保护和配置文件页面

[个人资料页面组件](https://github.com/auth0-blog/angular-auth0-aside/tree/master/src/app/profile)可以显示已验证的用户个人资料信息。但是，我们只希望该组件在用户登录时可访问。
[通过认证的API请求和登录/注销](https://github.com/auth0-blog/angular-auth0-aside/blob/master/src/app/home/home.component.ts)实施，最后一步是保护我们的个人资料路线免受未经授权的访问。 [`auth.guard.ts`路由守卫](https://github.com/auth0-blog/angular-auth0-aside/blob/master/src/app/auth/auth.guard.ts)可以检查身份验证并有条件地激活路由。守卫在[`app-routing.module.ts`文件](https://github.com/auth0-blog/angular-auth0-aside/blob/master/src/app/app-routing.module.ts)中的特定路由上实现，如下所示：

```js
// src/app/app-routing.module.ts
...
import { AuthGuard } from './auth/auth.guard';
...
      {
        path: 'profile',
        component: ProfileComponent,
        canActivate: [
          AuthGuard
        ]
      },
...
```

## 更多资源

我们拥有经过验证的Node API和Angular应用程序，包括登录，注销，个人资料信息和受保护的路线。要了解更多信息，请查看以下资源：

- [Why You Should Always Use Access Tokens to Secure an API](https://auth0.com/blog/why-should-use-accesstokens-to-secure-an-api/)
- [Navigating RS256 and JWKS](https://auth0.com/blog/navigating-rs256-and-jwks/)
- [Access Token](https://auth0.com/docs/tokens/access-token)
- [Verify Access Tokens](https://auth0.com/docs/api-auth/tutorials/verify-access-token)
- [Call APIs from Client-side Web Apps](https://auth0.com/docs/api-auth/grant/implicit)
- [How to implement the Implicit Grant](https://auth0.com/docs/api-auth/tutorials/implicit-grant)
- [Auth0.js Documentation](https://auth0.com/docs/libraries/auth0js)
- [OpenID Standard Claims](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims)

# 结论

在今天的文章中，我们比较了Cookie和基于令牌的身份验证之间的区别。我们强调了使用令牌的优势和关注点，并撰写了一个简单的应用程序来展示JWT在实践中的工作方式。使用令牌有很多原因，Auth0在这里确保实现令牌认证非常简单和安全。最后，我们引入了渐进式Web应用程序，帮助您的Web应用程序在移动设备上感受更原生。今天[注册](https://auth0.com/signup)一个免费帐户，并在几分钟内启动并运行。
