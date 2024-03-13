# OAuth2 Authentication with React

I'm unfortunately going to have to add my voice to some assorted tidbits I've
found regarding OAuth2 authentication in React apps. While there's been some
good efforts made, the reality is that it's a bit complicated (at least for
React beginners like myself) and has a fair number of moving parts. Let's start,
then, by identifying all the moving parts and trying to code each of them in
turn.

First, we have the React SPA itself. This is going to be relevant, because it's
the top level of our React app, and some of what we do is going to be done at
that level. Next, we have the BrowserRouter, which will mount our callback
component for us. Which leads us to the callback component itself. Finally, we
have a Login component that displays a login/logout button, as well as the
currently logged in user's email, if logged in.

## The React SPA

```typescript
const App: React.FC = () => {
    return (
        <ProvideAuth>
            <BrowserRouter>
                <div>
                    <div className="row"><Login /></div>
                    <Routes>
                        <Route element={<LoginCallback />} path="/callback" />
                        <Route element={<Home />} path="/" />
                    </Routes>
                </div>
            </BrowserRouter>
        </ProvideAuth>
    );
};
```

The application doesn't need much explanation. There's a home component (which
will simply display a greeting for now), a callback component which is
responsible for obtaining access tokens given authorization codes (OAuth2
Authorization Code Flow), and a Login component that displays the login/logout
button and logged-in username.

Before I go too much further, though, I want to jump in here with a word about
the `ProvideAuth` tag seen above. That's part of the custom hooks we use to
interact with the login server, so let's look at that before we go anywhere
else:

```typescript
import React, {createContext, useContext, useState} from "react";

// we are providing a React context of the following type:
interface AuthContextType {
    isAuthenticated: boolean,
    idToken: string | null
    accessToken: string | null,
    refreshToken: string | null,
    name: string | null,
    email: string | null,
    login: () => void,
    logout: () => void,
    processLoginResponse: (response: Response) => void,
    processUserInfoResponse: (response: Response) => void
}

// The default React context has no values and no functionality
const defaultAuthContext : AuthContextType = {
    isAuthenticated: false,
    idToken: null,
    accessToken: null,
    refreshToken: null,
    name: null,
    email: null,
    login: () => {},
    logout: ()=> {},
    processLoginResponse: (response: Response) => {},
    processUserInfoResponse: (response: Response) => {}
}

const AuthContext = createContext<AuthContextType>(defaultAuthContext);

// provides the `ProvideAuth` tag for our React app. Any child components will
// have access to the authentication context
function ProvideAuth({children}) {
    const auth = useProvideAuth();
    return (
        <AuthContext.Provider value={auth}>
            {children}
        </AuthContext.Provider>
    )
}

// the hook that child components will use to access the authentication context
function useAuth() {
    return useContext(AuthContext);
}

// the state-backed implementation of the authentication context
function useProvideAuth() {
    const [isAuthenticated, setAuthenticated] = useState(false);
    const [idToken, setIdToken] = useState("");
    const [accessToken, setAccessToken] = useState("");
    const [refreshToken, setRefreshToken] = useState("");
    const [name, setName] = useState("");
    const [email, setEmail] = useState("");

    // calls the auth endpoint of the realm and redirects back to the home URL
    const login = () => {
        const baseUrl = process.env.REACT_APP_BASE_URL;
        const clientId = process.env.REACT_APP_CLIENT_ID;
        const realmUrl = process.env.REACT_APP_REALM_URL;
        let q = [
            `redirect_uri=` + encodeURIComponent(`${baseUrl}/callback`),
            `scope=openid profile email`,
            `response_type=code`,
            `client_id=` + encodeURIComponent(`${clientId}`)
        ].join('&')
        window.location.replace(`${realmUrl}/protocol/openid-connect/auth?${q}`)
    }

    // calls the logout endpoint of the realm which will redirect back to the
    // home URL
    const logout = async () => {
        const realmUrl = process.env.REACT_APP_REALM_URL;
        await fetch(`${realmUrl}/protocol/openid-connect/logout`)
        // clear client-side login state
        setAuthenticated(false);
        setEmail("");
        setName("");
        setAccessToken("");
        setRefreshToken("");
        setIdToken("");
    }

    // process the response from the token endpoint using the authorization code
    const processLoginResponse = async (response: Response) => {
        let json = await response.json();
        setIdToken(json["id_token"]);
        setAccessToken(json["access_token"]);
        setRefreshToken(json["refresh_token"]);
        setAuthenticated(true);
    }

    // process the userinfo response to get name and email address
    const processUserInfoResponse = async (response: Response) => {
        let json = await response.json();
        setName(json["name"]);
        setEmail(json["email"]);
    }

    // return a compatible context object
    return {
        isAuthenticated,
        idToken,
        accessToken,
        refreshToken,
        name,
        email,
        login,
        logout,
        processLoginResponse,
        processUserInfoResponse
    }
}

export {useAuth, ProvideAuth}
```

There's a lot of individual methods there, but I added some comments, and none
of the methods are particularly complicated. But it's not necessary to entirely
understand the code to know how to use it effectively. There are two key points.
First, you'll need to call `useAuth()` from your React Component. This provides
the authentication context, which itself provides the following methods:

- `login`: redirects the user to the login page for the realm and handles the
  authorization code callback
- `logout`: signs the user out of the authentication server and redirects back
  to the home page
- `processLoginResponse`: used by the login callback to retrieve an access token
  from the authorization code
- `processUserInfoResponse`: used by the login callback to retrieve the user's
  name and email address
  Second, you need to wrap all React components that need the authentication
  context in the `ProvideAuth` tag, as shown in the initial example above. Only
  components whose ancestors include `ProvideAuth` can call `useAuth()`.

## The LoginCallback

The `LoginCallback` is responsible for receiving the authorization code in the
form of a callback URL. Once the code is received, it will be exchanged for id,
access, and refresh tokens. Those values will be stored in the authentication
context for later use, and then the userinfo endpoint will be queried to
complete the authentication context values.

```typescript
const LoginCallback : React.FC = () => {
    const location = useLocation();
    const navigate = useNavigate();
    const auth = useAuth();
    const [code, setCode] = useState("");
    const [lastCode, setLastCode] = useState("");

    const getToken = useCallback(async (code: string) => {
        let q = `code=` + encodeURIComponent(code) +
                `&redirect_uri=` + encodeURIComponent(`${process.env.REACT_APP_BASE_URL}/callback`) +
                `&client_id=` + encodeURIComponent(`${process.env.REACT_APP_CLIENT_ID}`) +
                `&client_secret=` + encodeURIComponent(`${process.env.REACT_APP_CLIENT_SECRET}`) +
                `&grant_type=authorization_code`;
        return await fetch(`${process.env.REACT_APP_REALM_URL}/protocol/openid-connect/token`, {
            method: "POST",
            headers: {"Content-Type": "application/x-www-form-urlencoded"},
            body: q
        });
    }, []);


    // this effect will update the code if it is new. The comparison to lastCode ensures that this
    // callback will only fetch one token per code, regardless of how many times this callback is rendered.
    useEffect(() => {
        let search = location.search;
        const theCode = (search.match(/code=([^&]+)/) || [])[1];
        if (theCode !== lastCode)
            setCode(theCode);
    }, [location.search, lastCode]);

    // this effect will fetch the token every time the code is updated
    useEffect(() =>
    {
        if (code) {
            getToken(code)
                .then(res => auth.processLoginResponse(res))
                .then(_ => navigate("/"))
                .catch(error => console.log(error))
            setLastCode(code);
        }
    }, [code, auth, getToken, navigate])


    return <div></div>;
}


export default LoginCallback;
```

I'll only note that there are two separate effects required to accomplish the
login. In the course of my testing, I discovered that it was not possible to do
the `lastCode` comparison within the same effect as the token fetch. Experienced
React users will understand that this is due to the asynchronous nature of
`useState()`. It was very frustrating to discover that I could not simply call
`setLastCode` and then immediately use `lastCode`'s new value. So, I update the
code if it's new, and then fetch a new token for every new code.

## The Login Component

The `Login` component will display a button for logging in or out, as well as
the currently logged in user's email address. This is illustrative, as it's
easily modified to suit your tastes. The most notable part of this component is
how it will retrieve userinfo every time the access token changes, and update
the component display accordingly. This demonstrates how to achieve
inter-component communication using React hooks: one component will update a
value within the context (`setAccessToken`, called from `processLoginResponse`),
and another will use that value as a dependency for its own effects.

```typescript
const Login: React.FC = () => {
    const auth = useAuth();

    const getUserInfo = useCallback(async () => {
        return await fetch(`${process.env.REACT_APP_REALM_URL}/protocol/openid-connect/userinfo`, {
            headers: { "Authorization": `Bearer ${auth.accessToken}` }
        });
    }, [auth.accessToken]);

    // this effect will populate the user details from the userinfo endpoint
    useEffect(() => {
        if (auth.accessToken) {
            getUserInfo()
                .then(res => auth.processUserInfoResponse(res))
                .catch(error => console.log(error));
        }
    }, [auth.accessToken, auth, getUserInfo]);

    if (auth.isAuthenticated)
        return (
            <div className="row">
                Logged in as {auth.email}
                <Button onClick={auth.logout}>Logout</Button>
            </div>
        );
    else {
        return <Button onClick={auth.login}>Login</Button>;
    }
}

export default Login;
```

## Conclusion

The above code is the verbatim implementation of an OAuth2 login component for
React. It demonstrates the use of React hooks and contexts to share state
between components and communicate changes between them. I suppose I should also
mention a couple of failed approaches. My first attempt used a Material Dialog
component to display the login within a modal iframe. However, this fails
because the LoginCallback component is then rendered within the iframe, and the
application loaded a second time. Another approach suggested similar, but
running the HTML through PurifyDOM first. This doesn't work for the Keycloak
server at all.

I hope you find this code useful. I am unfortunately not able to take on the
maintenance burden of releasing and maintaining this code as a resuable package.
If you're reading this and you want to, please feel free to do so. This code is
provided as-is without warranty or expectation of updates.

### Credits

I'd like to add a special thank you to [Param Singh](https://medium.com/@paramsingh_66174)
for his article on the subject, as well as Tasos Kakour (Github: @tasoskakour) for
his `useOAuth2` package on Github. Both of these authors were significant contributors to this article.
