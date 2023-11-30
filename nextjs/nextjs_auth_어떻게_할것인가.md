# nextjs에서 Token 관리

## 배경

nextjs에서 auth 관련처리는 역시 nextAuth가 대표적인 방법이다.
하지만 웹뷰를 보여주는 웹을 만들기위해서는 곤란하다.
cookie에 들어있는 값은 nextAuth를 통해서만 만들어진 암호화된 값들이다.

앱에서 쿠키를 셋팅해주면서 웹뷰로 전단하는데에 문제가 있다는 뜻이다.

이것 외에 문제가 또 있다.

nextauth discussions에 올라온 내용이다.

https://github.com/nextauthjs/next-auth/discussions/3550

axios instance에서 매번 token 값을 사용하려면 계속 nextjs 서버에 session을 가져와야되고 불필요한 서버요청이 많아 질 수 있다는 것이다.

야매로 이것저것 처리해볼 수 있지만 외부서버와 token으로 통신할 때는 최선의 방법이 아니라는 느낌을 많이 받는다.

위에 디스커션에도 결론에 다다르지 못했다.

서버요청 많을 수도 있다. 무시하고 넘어가도 된다. 하지만 내 가장 큰 문제는 앱과 웹 사이 쿠키 셋팅이다.

nextjs로 만들면 안되었을 수도 있다 하지만. 이미 엎지러진 물.

nextAuth를 버리자.

## 방법

결론 부터 말하면 흔한 로그인 방법이다. http Only 쿠키에 refreshToken을 두고 accessToken은 메모리에서 관리하는 것이다.

### http only 쿠키로 refreshToken 관리하기

http only 쿠키로 refreshToken을 관리하기 위해서는 서버와 맞춰야할 것들이 많다.

서버에서는 허용할 도메인을 설정해줘야하고, 서버와 클라이언트간의 도메인이 맞지 않으면 쿠키를 담아가며 통신할 수 없다.

하지만 우린 nextjs다.

refreshToken을 body response로 받아서 nextjs 서버에서 httpOnly 쿠키로 관리해줄 것이다.

### accessToken 메모리상에서 관리하기

메모리 어디에 둘것인가? 라는 문제가 있다.

일반 React App이라면 여러가지 상태관리 라이브러리 중에 하나를 선택했으면 될 것이다.
하지만 대부분의 상태관리 라이브러리는 useContext를 이용하여 구현되어 있다.

그리고 당연히 서버컴포넌트에서는 hook을 사용할 수 없다.

zustand는 observe를 이용하여 구현되어 있어서 state를 사용하지 않고 관리할 수 있다.

그렇기에 zustand를 선택하였다.

### 구현 시나리오

> 로그인

1. 로그인 api route를 사용하여 서버에서 받아온 refreshToken과 accessToken을 받아온다.
2. accessToken은 zustand store에 담아준다.

   - zustand authStore signin 함수

   실패 성공 여부에 따라 authStore를 업데이트 해준다.

   ```jsx
   signIn: async (body: { email: string, password: string }) => {
     try {
       const signInResponse = await signIn(body);

       const response = {
         isSuccess: true,
         data: {
           accessToken: signInResponse.accessToken,
           userId: signInResponse.userId,
         },
       };

       set(response.data);
       return response;
     } catch (err) {
       if (axios.isAxiosError(err)) {
         const response = {
           isSuccess: false,
           data: null,
           err: err.response?.data,
         };
         return response;
       }

       return {
         isSuccess: false,
         data: null,
       };
     }
   };
   ```

3. refreshToken은 cookie에 넣어준다.

   - sign-in api route

   ```jsx
   export async function POST(req: NextRequest) {
     const data = await req.json()

     try {
       const res = await axiosInstance
         .post<{
           data: { accessToken: string; refreshToken: string; userId: number }
         }>('/authentication/login', data)
         .then(v => v)

       const nextResponse = NextResponse.json(res.data)
       nextResponse.cookies.set({
         name: 'refreshToken',
         value: res.data.data.refreshToken,
         httpOnly: true,
       })
       return nextResponse
     } catch (err) {
       if (!axios.isAxiosError(err)) {
         return NextResponse.json({ message: 'failed login' }, { status: 500 })
       }
       return NextResponse.json(
         { message: err.response?.data.message },
         {
           status: err.response?.data.code,
         }
       )
     }
   }
   ```

> 로그아웃

http only cookie를 지워주기 위해 api route를 사용해야 한다.

redirect는 서버 액션시 작동하고

csr에서 요청시는 처리하는곳에서 api 응답후 직접 클라이언트쪽에서 navigate를 해줘야한다.

- signout api route

```tsx
import { cookies } from "next/headers";

import { redirect } from "next/navigation";

export async function POST() {
  //@ts-ignore
  cookies().delete("refreshToken");

  redirect("/auth/email");
}
```

- authStore signOut

```jsx
signOut: () => {
        axios.post("/api/auth/sign-out");
        set({ accessToken: null, userId: null });
        return;
      },
```

> 새로고침 or 재접속

1. cookie에서 refresh token을 가져온다.
2. refreshToken으로 accessToken을 가져온다.
3. accessToken을 가져오지 못하면 로그아웃 처리를 해준다.

- refreshToken api route

```
export async function PUT(req: NextRequest) {
  const refreshToken = req.cookies.get('refreshToken')

  if (!refreshToken) {
    req.cookies.delete('refreshToken')
    redirect('/')
  }

  try {
    const res = await axios
      .put<{
        data: { accessToken: string; refreshToken: string; userId: number }
      }>(
        `${process.env.NEXT_PUBLIC_API}/authentication/refresh`,
        {},
        {
          headers: {
            Authorization: `Bearer ${refreshToken.value}`,
            'Content-Type': 'application/json',
          },
        }
      )
      .then(v => v)

    return NextResponse.json(res.data)
  } catch (err) {
    //@ts-ignore
    cookies().delete('refreshToken')
    redirect('/')
  }
}
```

- authStore refreshAccessToken

```jsx
refreshAccessToken: async () => {
  try {
    const res = await getAccessToken();
    console.log(res);
    if (!res.accessToken) {
      const response = {
        accessToken: null,
        userId: null,
      };
      set(response);
      return response;
    }
    const response = {
      accessToken: res.accessToken,
      userId: res.userId,
    };
    set(response);
    return response;
  } catch (err) {
    const response = {
      accessToken: null,
      userId: null,
    };
    set(response);
    return response;
  }
};
```

### 보안 취약점

위에 방법으로 관리하는 이유는 아래 보안 취약점들 때문이다.

> XSS (Cross site scripting) 취약점

나쁜녀석들이 input 등에 자바스크립트 코드를 실행시켜서 로컬스토리지나 쿠키에 저장된 Token에 접근하거나 할 수 있다.

> CSRF (Cross-site Request Forgery) 공격

나쁜녀석들 서버에 요청을 시켜서 쿠키에 있는 accessToken값에 접근하여 나쁜녀석들 서버에서 accessToken를 알아낼 수 있다.

그렇기에 http only 를 사용해서 refreshToken cookie 값을 이용하고
accessToken은 메모리에 둔다.

https://docs.pmnd.rs/zustand/integrations/persisting-store-data#hydration-and-asynchronous-storageshttps://stackoverflow.com/questions/65160156/remove-cookies-and-sign-out-server-side-in-next-js

[https://velog.io/@yaytomato/프론트에서-안전하게-로그인-처리하기](https://velog.io/@yaytomato/%ED%94%84%EB%A1%A0%ED%8A%B8%EC%97%90%EC%84%9C-%EC%95%88%EC%A0%84%ED%95%98%EA%B2%8C-%EB%A1%9C%EA%B7%B8%EC%9D%B8-%EC%B2%98%EB%A6%AC%ED%95%98%EA%B8%B0)
