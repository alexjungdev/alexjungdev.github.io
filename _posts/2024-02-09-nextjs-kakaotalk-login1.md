---
title: "[Next.js] Kakaotalk 로그인 구현 & 파이어베이스 연결 (1)"
date: 2024-02-09 16:51:00+0900
categories: [Front-End,Next.js]
tags: [front-end,next,js,.env]     # TAG names should always be lowercase
---
## **개발 배경**
웹사이트에 파이어베이스를 이용한 구글 소셜 로그인을 구현하였다. 이 과정은 매우 간단하였기 때문에 크게 공부를 하거나 할 필요가 없
었다. 하지만 카카오톡을 이용한 로그인은 생각보다 꽤 많이 복잡했다. 개발하는 과정에서도 우열곡절이 많았고 다시 구현할때 좀 더 쉽게 구현할 수 있도록 하기 위하여 이렇게 글로 정리하게 되었다. 여기서 소셜 로그인 구현 및 그 과정에서 발생한 에러들과 해결법을 전부 작성해보겠다.
<br>
<br>

## **로그인 알고리즘**
아래의 알고리즘대로 차근차근 한번 만들어보자(글을 보시는 분들이 원하는대로 알고리즘을 수정하고 바꾸셔도 됩니다).
난 Rest API와 JavaScript 중에서 JavaScript 로그인 방식으로 개발할 것이다.

1. 카카오 개발자 등록 & 설정
2. 카카오톡 로그인 버튼 생성
3. 카카오톡 로그인창으로 리디렉트
4. 권한 선택 이후 로그인 완료
5. 302 Redirect로 인가 코드 전달
6. (Back-End) 토큰 요청 & 토큰 발급
7. (Back-End) 토큰을 통해 유저 정보 불러오기
8. (Back-End) Firebase Functions를 이용하여 유저 정보로 Custom Login

## **카카오 개발자 등록 & 설정**
### **1. 카카오 개발자 등록 & 앱 생성**
![Kakao Developer1](/assets/img/kakao1.png)
Kakao Developer로 가서 개발자 등록을 하고 사용할 애플리케이션을 추가한다(애플리케이션에는 Android, Ios, Web 전부 있다).
<br>
<br>

### **2. 플랫폼 등록 & 도메인 설정**
![Kakao Developer2](/assets/img/kakao2.png)
![Kakao Developer3](/assets/img/kakao3.png)
내가 만들 애플리케이션에 해당하는 플랫폼을 등록하고 도메인을 설정한다. 도메인은 추후에 배포하고나서도 수정이 가능하니 일단은 내가 작업할 localhost로 설정해주었다.
<br>
<br>

### **3. 비즈 앱 전환 & 권한 설정**
![Kakao Developer4](/assets/img/kakao4.png)
![Kakao Developer5](/assets/img/kakao5.png)
유저들이 카카오톡으로 로그인을 할때 이메일을 필수로 받기 위해 권한을 설정해야한다. 하지만 기본적으로는 프로필 권한만 설정할 수 있다. 이메일 권한도 설정할 수 있게 하기위해 **개인 개발자 비즈 앱 전환**을 눌러준다. 그러면 **카카오계정(이메일)**을 필수 동의할 수 있게된다.
<br>
<br>

### **4. Redirect URL 설정**
![Kakao Developer Redirect](/assets/img/kakao_redirect.png)
이메일 권한도 설정하였다면 이제 로그인 이후 리디렉트를 하기위한 설정을 해준다.
나같은 경우엔 app/kakao 폴더를 만들고 app/kakao에 리디렉트 페이지를 만들기 위하여 사진처럼 설정을 했다.

## **카카오톡 로그인 버튼**
![Kakao Developer Login But](/assets/img/kakao_login_but.png)
이렇게 로그인 카카오톡 로그인 버튼을 만들어보았다. 아래에 있는 구글 로그인과 커스텀 로그인은 따로 구현을 하였다.
```
<button
    className="auth-btn yellow"
        onClick={SignIn_Kakao}>
        <RiKakaoTalkFill size={10} color={"white"} className="auth-btn-icon" />
        <text className="text-small auth-btn-text font-bold tracking-tighter text-white">
            카카오톡으로 시작하기
        </text>
</button>


const SignIn_Kakao = async () => {
        const Kakao = window.Kakao;
        if (Kakao && !Kakao.isInitialized()) {
            Kakao.init(process.env.NEXT_PUBLIC_KAKAO_JAVASCRIPT_KEY);
        }
        try {
            const redirectUri = `${process.env.NEXT_PUBLIC_KAKAO_REDIRECT_URI}`;

            await window.Kakao.Auth.authorize({
                redirectUri,
            });
        }
        catch (error) {
            throw error;
        }

    }
```
버튼은 이런 코드로 구현했다. Tailwind CSS를 사용했고 카카오톡 아이콘은 React Icons를 사용했다.
참고로 카카오의 JS KEY 와 REDIRECT URI는 토큰때문에 노출되어도 괜찮다고 한다(https://devtalk.kakao.com/t/rest-api/128734). 따라서 저렇게 NEXT_PUBLIC으로 설정해주었다.

## 카카오톡 로그인 리디렉트
![Kakao Developer Redirect 2](/assets/img/kakaotalk_login_redirect_open.png)
버튼을 클릭하면 이렇게 리디렉트 창이 뜬다. 나는 이전에 로그인을 해서 이미 계정이 저장되어있어 이렇게 뜨지만 아마도 보통은 권한을 선택하는 창이 뜰 것이다.

## 권한 선택 이후 로그인 완료
![Kakao Auth Token](/assets/img/kakaotalk_code.png)
권한을 선택하시고 로그인을 완료하면 이제 끝난다. app/kakao(App Router이 아닌 Pages Router 사용시엔 page/kakao)에 page.js를 따로 만들어놔서 localhost/kakao로 code가 붙은채로 이동된다.
여기에는 로딩바나 스피너와 같은 로딩에 사용되는 창을 추가해놓아야 한다. 왜냐면 여기서 부터 이제 밴엔드인 Firebase Function으로 들어가기 때문이다.

Firebase Function부턴 다음 게시물에 작성해보겠다.


