```
//userAuthAction.ts
'use server';

import { cookies } from 'next/headers';

export async function loginCookie() {
  cookies().set({
    name: 'isLogin',
    value: 'true',
    httpOnly: true,
    path: '/',
  });
}

```

이런 느낌으로 서버 액션을 실행해서 쿠키를 셋팅해주는 함수를 만들 수도 있음
