# [HTTP 系列] 第 X 篇 —— 聊一聊 JWT

> 半年前我在写个人博客的 [后台管理系统](https://github.com/Yancey-Blog/BLOG_CMS) 时，用到了 JWT 做登录凭证。当时因为某些原因，仅仅是使用它完成了想要的效果，而不懂它的原理是什么。这次适逢写 HTTP 系列，索性深入研究一下 JWT 的原理，也希望能帮到大家。

## 什么是 JWT

首先，JWT 不是 **劲舞团**，也不是 **今晚她** (emmmmmm，好邪恶)。

JWT (JSON Web Tokens) 是一个开放的，遵循 [RFC 7519](https://tools.ietf.org/html/rfc7519) 行业标准的方法，用于双方传递安全可靠的信息。简言之，它就是一串 **token** 字符串，如果你的请求携带正确的 token，服务端就认为你的请求是安全的。
