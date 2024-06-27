---
title: React Axios Interceptor로 refresh token 재발급 로직 구현
categories:
  - react
tags:
  - react
  - jwt
  - axios
---
# Axios Interceptor
이번 글을 저번 [React JWT 로그인 구현](https://codevarty.github.io/react/react-jwt-1/) 에 이어지는 내용입니다. 

저번 시간에는 axios를 통해 로그인을 하면 해당 토큰을 localstorage에 저장하는 것으로 로그인하는 방법에 대해 알게 되었다. 이번에는 **Access Token이 만료 되었을 때 Refresh Token을 재발급 요청**을 보내는 방법에 대해 알아보고자 한다.

## Refresh Token 재발급 로직
먼저 로그인을 통해 Access Token과 Refresh Token을 서버에서 발급 받아 클라이언트가 저장을 한다. 그리고 API 요청을 할 때 발급 받은 Access Token을 사용하게 된다. 

Access Token은 보안 상 탈취 위험을 줄이기 위해 만료시간을 짧게 설정을 하게 된다. 그래서 클라이언트가 서버로 부터 **만료 응답 코드를 받으면** 자동으로 Refresh Token을 사용하여 서버에 토큰 재발급 요청을 보내게 된다.

그러나 만료 토큰 이외에 401 응답 코드를 받으면 로그아웃을 시킨다.

## Axios Interceptor 작성
인터셉터를 사용하기 위해 Axios 인스턴스를 생성한다. 인터셉터는 요청 인터셉터와 응답 인터셉터로 두 가지가 있다.

먼저 `api.js` 파일을 생성을 한다.

먼저 전체 코드를 보여준다.

```js
import axios from 'axios'

const api = axios.create() // axios 인스턴스 생선
  
export default api;  
  
api.interceptors.request.use(  
    (config) => {  
        const accessToken = localStorage.getItem("accessToken");  
  
        if (!accessToken) {  
            // access token 이 없는 경우 로그인 페이지로 이동  
            window.location.href = '/login';  
            return config;  
        }  
  
        config.headers['Authorization'] = `Bearer ${accessToken}`;  
        return config;  
    },  
    error => {  
        return Promise.reject(error);  
    }  
)  
  
api.interceptors.response.use(  
    (response) => {  
        if (response.status === 400) {  
            console.log("404 error");  
        }  
        return response;  
    },  
    async (error) => {  
        const {config, response: {status, data}} = error  
        // 잘못된 토큰이 었을 경우 로그아웃
        if (status === 401 && data === "InvalidToken") {  
            logout()  
        }  
  
        if (status === 401 && data === "ExpiredToken") {  
            try {  
                const tokenRefreshResult = await api.post('/api/refresh-token', null, {  
                    headers: {  
                        'Content-Type': 'application/json',  
                        'Authorization-refresh': `Bearer ${localStorage.getItem('refreshToken')}`  
                    }  
                });  
                if (tokenRefreshResult.status === 200) {  
                    const accessToken = tokenRefreshResult.headers.get('Authorization');  
                    const refreshToken = tokenRefreshResult.headers.get('Authorization-refresh');  

					// 토큰 재바급이 잘못된 경우 로그아웃 시킨다.
                    if (!accessToken || !refreshToken) {  
                        logout();  
                        return                    }  
                    // 새로 발급받은 토큰을 스토리지에 저장  
                    localStorage.setItem('accessToken', accessToken);  
                    localStorage.setItem('refreshToken', refreshToken);  
                    // 토큰 갱신 성공. API 재요청  
                    return api(config)  
                } else {  
                    logout();  
                }  
            } catch (e) {  
                logout();  
            }  
        }  
  
        return Promise.reject(error);  
    }  
)  
  
function logout() {  
    localStorage.clear()  // localstroage를 비운다.
}
```

### 요청 인터셉터
```js
api.interceptors.request.use(  
    (config) => {  
        const accessToken = localStorage.getItem("accessToken");  
  
        if (!accessToken) {  
            // access token 이 없는 경우 로그인 페이지로 이동  
            window.location.href = '/login';  
            return config;  
        }  

		// access token 을 보낼 때 'Bearer ' 해더를 붙인다.
        config.headers['Authorization'] = `Bearer ${accessToken}`;  
        return config;  
    },  
    error => {  
        return Promise.reject(error);  
    }  
)  
```

요청 인터셉터에서 Access Token을 `localstroage`에서 가져와 서버에 API 요청을 보낼 때 header에 함께 보낼 수 있도록 코드를 자성하였다.

이 때 `localstroage` 안에 토큰이 없는 경우 로그인이 되어 있지 않다고 생각하여 로그인 페이지로 이동을 시킨다.

### 응답 인터셉터
```js
api.interceptors.response.use(  
    (response) => {  
        if (response.status === 400) {  
            console.log("404 error");  
        }  
        return response;  
    },  
    async (error) => {  
        const {config, response: {status, data}} = error  
        // 잘못된 토큰이 었을 경우 로그아웃
        if (status === 401 && data === "InvalidToken") {  
            logout()  
        }  
  
        if (status === 401 && data === "ExpiredToken") {  
            try {  
                const tokenRefreshResult = await api.post('/api/refresh-token', null, {  
                    headers: {  
                        'Content-Type': 'application/json',  
                        'Authorization-refresh': `Bearer ${localStorage.getItem('refreshToken')}`  
                    }  
                });  
                if (tokenRefreshResult.status === 200) {  
                    const accessToken = tokenRefreshResult.headers.get('Authorization');  
                    const refreshToken = tokenRefreshResult.headers.get('Authorization-refresh');  

					// 토큰 재바급이 잘못된 경우 로그아웃 시킨다.
                    if (!accessToken || !refreshToken) {  
                        logout();  
                        return                    }  
                    // 새로 발급받은 토큰을 스토리지에 저장  
                    localStorage.setItem('accessToken', accessToken);  
                    localStorage.setItem('refreshToken', refreshToken);  
                    // 토큰 갱신 성공. API 재요청  
                    return api(config)  
                } else {  
                    logout();  
                }  
            } catch (e) {  
                logout();  
            }  
        }  
  
        return Promise.reject(error);  
    }  
)  

// 로아웃하는 경우 토큰을 비울 수 있도록 한다. 
function logout() {  
    localStorage.clear()  // localstroage를 비운다.
}
```

응답 인터셉터에서 error 부분이 중요하다. 여기에서 토큰 만료 로직을 작성한다.

```js
if (status === 401 && data === "ExpiredToken") {...}
```

백엔드에서 토큰이 만료된 경우 응답 데이터로 "ExpiredToken"을 보내게 된다. 

이 때 클라이언트에서 재발급 로직을 작성한다.

```js
        if (status === 401 && data === "ExpiredToken") {  
            try {  
                const tokenRefreshResult = await api.post('/api/refresh-token', null, {  
                    headers: {  
                        'Content-Type': 'application/json',  
                        'Authorization-refresh': `Bearer ${localStorage.getItem('refreshToken')}`  
                    }  
                });  
                if (tokenRefreshResult.status === 200) {  
                    const accessToken = tokenRefreshResult.headers.get('Authorization');  
                    const refreshToken = tokenRefreshResult.headers.get('Authorization-refresh');  

					// 토큰 재바급이 잘못된 경우 로그아웃 시킨다.
                    if (!accessToken || !refreshToken) {  
                        logout();  
                        return                    }  
                    // 새로 발급받은 토큰을 스토리지에 저장  
                    localStorage.setItem('accessToken', accessToken);  
                    localStorage.setItem('refreshToken', refreshToken);  
                    // 토큰 갱신 성공. API 재요청  
                    return api(config)  
                } else {  
                    logout();  
                }  
            } catch (e) {  
                logout();  
            }  
        }  
```

 이번 글에서는 토큰을 재발급하는 API를 통해 Access Token 및 Refresh Token 을 재발급하도록 한다.

재발 급을 받기 위해 Refresh Token을 보내 해당 토큰을 검증한 다음 서버에서 토큰을 재발급하는 식이다.

```js
function logout() {  
    localStorage.clear()  // localstroage를 비운다.
}
```

서버에서 예외 응답을 받거나 잘못된 토큰을 보낸 경우 로그아웃을 시키도록 한다.

이번 글에서는 간단하게 `localstroage`의 토큰을 모두 지우게 해놓았다.

# 결론
이번 글에서는 Refesh Token을 재발급 할 수 있게 Axios에 인터셉터 로직을 작성하였다.

클라이언트에서 재발급하는 원리에 대해 알게 되었고 Axios 에 인터셉터라는 기능에 대해 알게 되었다.