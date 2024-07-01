# Installation

- Git clone this [project](https://github.com/WildCodeSchool-2024-02/2023-09-JS-Paris-jobApp)
- Switch to the authorization branch `git checkout authorization`
- Run `npm install`
- Setup your .env file in both `server` and `client` folder
- Run `npm run db:migrate`
- Run `npm run dev:server`

# Your mission

The objective of this workshop is to set up a refresh token system to keep the connection active when refreshing your react application

## Step 1 : server configuration

### cookie setup

We must set up the cookie-parser middleware to allow us to read the cookies sent to the server

- Install `cookie-parser` package in the `server` folder
- In your `config.js` uncomment this line : 

```js
const cookieParser = require("cookie-parser");

app.use(cookieParser());
```

### cors setup 

to have access to our authorization header from our client we must modify the cors configuration to allow reading

- Update cors configuration in your `config.js` with the following : 

```js
const cors = require("cors");

app.use(
  cors({
    exposedHeaders: ["Authorization"],
    origin: [
      process.env.CLIENT_URL, // keep this one, after checking the value in `server/.env`
    ],
    credentials: true,
  })
);
```

We added here the `exposedHeaders` option to allow our Authorization header to be read from the client and the `credentials` option that allow us to send/retrieve cookie to/from the client.

## Step 2 : Generate refresh token & cookie

- First, on your login controller Generate a refresh token with one day of validity :

```js
const refreshToken = jwt.sign(
  { id: user.id, role: user.role },
  process.env.APP_SECRET,
  {
    expiresIn: "1d",
  }
);
```

- Then, send the refresh token as a cookie to the client :

```js
res
.cookie("refreshToken", refreshToken, {
  httpOnly: true,
  sameSite: "none",
})
.header("Authorization", token)
.json(user);
```
The `httpOnly` option to restrict the read of the cookie from the client
{:.alert-info}

The `sameSite` option allow us to send cookie to a client from another domain
{:.alert-info}

## Step 3 : Create a refresh route to re-generate access token

- Create a refresh method in your `authAction` : 

```js
const refresh = async (req, res, next) => {
  try {
    
  } catch (error) {
    return next(error);
  }
};

module.exports = { register, login, refresh, logout };
```

- In your refresh method, first get the `refreshToken` from the cookie :

```js
const { refreshToken } = req.cookies;
if (!refreshToken) {
  return res.status(401).send("Access Denied. No refresh token provided.");
}
```

- Verify the validity of the refreshToken and generate a new access token : 

```js
const decoded = jwt.verify(refreshToken, process.env.APP_SECRET);
const accessToken = jwt.sign({ id: decoded.id, role: decoded.role }, process.env.APP_SECRET, {
  expiresIn: "1h",
});
```

- Get back the user information using the id stored in the refreshToken : 

```js
const [[user]] = await tables.user.read(decoded.id);
delete user.password;
```

- Finally, send the new generated token to the client : 

```js
return res.header("Authorization", accessToken).json(user);
```

- Link your middleware to a new get route `/refresh`

## Step 4 : Create a logout route to remove the cookie 

- In your `authAction` add the logout method as following : 

```js
const logout = async ({res}) => {
  res.clearCookie("refreshToken").sendStatus(200);
}
```

- Link your middleware to a new get route `/logout`

## Step 5 : Implement refreshToken in your authContext

- In your auth context located in `App.jsx` add a new state isLoading :

```js
const [isLoading, setIsLoading] = useState(true);
```

- Then, add a useEffect hook with a `getAuth` method : 

```js
  useEffect(() => {
    const getAuth = async () => {
      try {
        
      } catch (error) {
        toast.error("Une erreur est survenue");
        setIsLoading(false);
      }
    }
    getAuth();
  }, []);
```

- In your `getAuth` method fetch the refresh routes : 

```js
const response = await fetch(
  `${import.meta.env.VITE_API_URL}/auth/refresh`,
  { credentials: "include" }
);
```
Don't forget to add the credentials option to allow the client to send the cookie
{:.alert-info}


- Then, if the response is okay, get the result (token, user) and set your authContext states : 

```js
 const token = response.headers.get("Authorization");
const user = await response.json();
setAuth({ isLogged: true, user, token });
```

- Whatever if the response is ok, set the loading state to false after the request is done :

```js
setIsLoading(false);
```

## Final step : Create a PrivateRoute component to restrict access to your page

- In your main create the PrivateRoute component as following : 

```js
const PrivateRoute = ({children}) => {
  const { auth, isLoading } = useOutletContext();
  const navigate = useNavigate();

  useEffect(() => {
    if (!isLoading) {
      if (!auth.isLogged) navigate("/login");
    }
  }, [auth, children, isLoading, navigate])

  if (!isLoading && auth.isLogged) return children;
  return "...loading";
}
```

Inspect the given code, if you don't understand it call your trainer !
{:.alert-info}

- Implement your PrivateRoute component to your restricted routes : 

```js
{
  path: "/",
  element: (
    <PrivateRoute>
       <Home />
    </PrivateRoute>
  ),
},
```

# The final word

We were able to set up a refresh token system to keep an active connection even when we refresh our application. 

However, this is not perfect, we still need to deal with the case where the token expires due to its validity period to dynamically call the refresh route to retrieve a new token in a transparent way for the user. 

Yes, as developers we must think about all the use cases and implement increasingly complex systems to find a balance between security and user experience. 

You're not done learning, this is just the beginning!