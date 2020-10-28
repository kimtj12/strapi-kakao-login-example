# Strapi - Kakao Login

본 문서는 Strapi CMS에 카카오 로그인을 연동하는 방법을 설명합니다.



## 카카오 어플리케이션 등록
1. https://developers.kakao.com 에서 회원가입 및 '내 어플리케이션' 생성
2. 앱설정>요약정보에서 '플랫폼' 설정
3. Web 플랫폼에서 도메인 등록 (localhost 가능)
4. 제품 설정>카카오 로그인에서 '카카오 로그인 활성화' 및 Redirect URL 등록
5. 카카오 로그인>동의항목에서 필요한 정보들을 추가 (예시에서는 닉네임과 이메일만 추가함)
6. 요약 정보의 REST API 키와 카카오 로그인>보안의 Client Secret 을 나중에 입력
7. Redirect URL 의 경우 기본 세팅의 경우 http://localhost:1337/connect/kakao/callback 로 등록



## Strapi 설정
기본 개념은 Strapi Admin 패키지 위에 카카오 로그인 커스텀 소스를 얹은 코드를 덮어씌우기 하는 과정으로 진행됨.
따라서 필요한 파일들을 불러(복사해)와서 필요한 카카오 로그인 코드 정보를 넣고서 Admin을 새로 빌드하면 됨.
**해당 파일(불러오는)의 내용은 버전별로 다를 수 있으니 꼭 자신의 버전에 맞는 소스를 구해서 복사 후 붙여넣기 필요**



### 1. 필요한 파일 생성

각 프로젝트의 루트에 extensions 폴더가 있습니다. 그 폴더 아래에 아래 경로들(폴더들)을 만든 후 빈 js 파일들을 생성해줍니다.

```text
extensions/users-permissions/services/Providers.js
extensions/users-permissions/config/functions/bootstrap.js
extensions/users-permissions/config/users-permissions-actions.js
extensions/users-permissions/admin/src/components/PopUpForm/index.js
extensions/users-permissions/admin/src/translations/en.json
```

이제 만들어진 빈 파일들에 내용을 채워 넣습니다. 우선 기존 소스를 가져와서 붙여넣고 그 다음으로 카카오 로그인을 위한 코드들을 추가하는 단계로 진행될 예정입니다.



### 2. 기존 소스 복사

기존 소스의 경우 Strapi Github 에서 복사할 수 있습니다. 그 소스는 아래 링크에 있습니다.

https://github.com/strapi/strapi/tree/master/packages/strapi-plugin-users-permissions

이곳에서 각 폴더에 맞는 곳으로 들어가서 코드를 복사해주세요.

예시)

```bash
extensions/users-permissions/services/Providers.js
```

는 리포지토리에서 

```
strapi/packages/strapi-plugin-users-permissions/services/Providers.js 
```

입니다.

예시와 같은 방법으로 상기 4개 파일을 생성/복사해주세요.



### 3. Providers.js 수정

`switch (provider` 안에 아래 case를 추가해주세요.

```javascript
case "kakao": {
      const kakao = purest({
        provider: "kakao",
        config: {
          kakao: {
            "https://kapi.kakao.com": {
              __domain: {
                auth: {
                  auth: { bearer: "[0]" },
                },
              },
              "[version]/{endpoint}": {
                __path: {
                  alias: "__default",
                  version: "v2",
                },
              },
            },
            "https://kauth.kakao.com": {
              "oauth/{endpoint}": {
                __path: {
                  alias: "oauth",
                },
              },
            },
          },
        },
      });
  
    kakao
    .query()
    .get("user/me")
    .auth(access_token)
    .request((err, res, body) => {
      if (err) {
        callback(err);
      } else {
        callback(null, {
          username: body.id,
          email: body.kakao_account.has_email
            ? body.kakao_account.email
            : `${body.id}@kakao.com`,
        });
      }
    });
  break;
}
```

여기서 주의할 점은 아래쪽에 callback(프론트로 정보를 넘겨주는 부분)인데 받아오는 항목에 따라서 body의 내용이 달라질 수 있다는 것입니다. 카카오 계정의 경우 이메일이 없을 수도 있어서 has_mail 값으로 체크를 해주고, 없는 경우 id값으로 이메일을 임의로 생성해줬습니다.



### 4. bootstrap.js 수정

grantConfig 오브젝트 안에 해당 내용을 추가해줍니다.

```js
 kakao: {
      enabled: false, // make this provider disabled by default
      icon: "kaggle", // The icon to use on the UI
      key: "", // our provider app id (leave it blank, you will fill it with the content manager)
      secret: "", // our provider secret key (leave it blank, you will fill it with the content manager)
      callback: "/auth/kakao/callback", // the callback endpoint of our provider
      scope: [
        // the scope that we need from our user to retrieve information
        "profile",
        "account_email",
      ],
    },
```
scope의 경우 받아오시려는 항목들에 맞게 수정해서 사용해주세요. 주석에 단 것 처럼 이전 카카오 개발자 페이지에서 받은 키와 시크릿 키는 나중에 어드민 페이지에서 넣을 수 있습니다. (빈칸으로 두셔도 됩니다.)



### 5. PopUpForm/index.js 수정

`switch (this.props.dataToEdit)` 안에 아래 case 를 추가해주세요.

```js
case "kakao":
        return `${strapi.backendURL}/connect/kakao/callback`;
```



### 6. en.json 수정

json 오브젝트 안에 아래 내용을 추가해주세요. 

```
'PopUpForm.Providers.kakao.providerConfig.redirectURL': 'The redirect URL to add in your Kakao application configurations',
```

뒤쪽에 문구는 각자 취향대로 입력하셔도 됩니다. (어드민 페이지에 보여질 문구입니다.)



### 7. Admin 페이지 새로 빌드

프로젝트 루트에서 `npm run build`  를 실행하여 새로운 어드민 페이지를 빌드해주세요. 다시 어드민을 실행해서 들어가보면 카카오가 추가되어 있습니다. 이제 맨 처음에 발급받은 키와 시크릿키를 입력하고 프론트엔드에서 쓸 Redirect URL을 넣고 실행하면 됩니다.



### 그 외 참고

- users-permissions-actions.js 파일은 별도의 수정이 필요하지 않습니다. 다만 bootstrap 참조 때문에 추가했습니다.
- 카카오 로그인을 실행하기 위해서는 http://localhost:3000/connect/kakao 으로 호출을 하게 되고 결과 값은 http://localhost:3000/connect/kakao/redirect 로 들어오게 됩니다.
- 상기 localhost:3000은 프론트엔드 상황에 따라서 달라질 수 있습니다. 
- 자세한 사항은 코드를 참고해주세요.
- 실제 production 환경에서는 해당 서버의 정확한 URL을 입력하고 카카오쪽에도 등록해야됩니다.

`config/server.js` 에서

```js
module.exports = ({ env }) => ({
  host: env('HOST', '0.0.0.0'),
  port: env.int('PORT', 1337),
  url: env('', 'http://localhost:1337'),
});
```

중에 http://localhost:1337 을 수정하시면 됩니다.



#### 이해되지 않는 부분이 있다면 kimtj@me.com 으로 문의주세요.