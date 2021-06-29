---
title: "FullStack React & Django Authentication : Django REST ,TypeScript, Axios, Redux & React Router"
date: 2021-06-29T03:11:05+01:00
draft: false
author: "Mangabo Kolawole"
featured_image: "djangoreact.png"
---

As a full-stack developer, understand how to build an authentication system with backend technology and manage the authentication flow with a frontend technology is crucial.

In this tutorial, we'll together build an authentication system using React and Django.
We'll be using Django and Django Rest to build the API and create authentication endpoints. And after, set up a simple login and profile page with React and Tailwind, using Redux and React router by the way.

## Backend

First of all, let's set up the project. Feel free to use your favorite python environment management tool. I’ll be using `virtualenv` here.

```shell

virtualenv --python=/usr/bin/python3.8 venv
source venv/bin/activate
```

- And after that, we install the libraries we’ll be using for the development and create the project.
  
```shell

pip install django djangorestframework djangorestframework-simplejwt

django-admin startproject CoreRoot
```

- We'll first create an app that will contain all the project-specific apps.
  
```shell
django-admin startapp core
```

- After the creation, delete all files and folders except `__init__.py` and `apps.py`.
- Then open the settings file containing Django configurations and add `core` to the INSTALLED_APPS :

```python
    # CoreRoot/settings.py
    ...
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    'core'
```

We can now create the user application and start adding features.

```shell
cd core && python ../manage.py startapp user
```

```python
    # CoreRoot/settings.py
    ...
    'rest_framework',
    
    'core',
    'core.user'
```

For this configuration to work, you'll need to modify the name of the app in core/user/apps.py

```python
# core/user/apps.py
from django.apps import AppConfig
    
    
class UserConfig(AppConfig):
    name = 'core.user'
    label = 'core_user'
```

And also the `__init__.py` file in `core/user` directory. 

```python
# core/user/__init__.py
default_app_config = 'core.user.apps.UserConfig'
```

## Writing User logic

Django comes with a built-in authentication system model which fits most of the user cases and is very safe. But most of the time, we need to do rewrite it to adjust the needs of our project. You may add others fields like bio, birthday, or other things like that.

### Creating a Custom User Model Extending AbstractBaseUser

A Custom User Model is a new user that inherits from `AbstractBaseUser`. But we’ll also rewrite the `UserManager` to customize the creation of a user in the database.
But it’s important to note that these modifications require special care and updates of some references through the `settings.py`.

```python
# core/user/models.py
from django.db import models

from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin


class UserManager(BaseUserManager):

    def create_user(self, username, email, password=None, **kwargs):
        """Create and return a `User` with an email, phone number, username and password."""
        if username is None:
            raise TypeError('Users must have a username.')
        if email is None:
            raise TypeError('Users must have an email.')

        user = self.model(username=username, email=self.normalize_email(email))
        user.set_password(password)
        user.save(using=self._db)

        return user

    def create_superuser(self, username, email, password):
        """
        Create and return a `User` with superuser (admin) permissions.
        """
        if password is None:
            raise TypeError('Superusers must have a password.')
        if email is None:
            raise TypeError('Superusers must have an email.')
        if username is None:
            raise TypeError('Superusers must have an username.')

        user = self.create_user(username, email, password)
        user.is_superuser = True
        user.is_staff = True
        user.save(using=self._db)

        return user


class User(AbstractBaseUser, PermissionsMixin):
    username = models.CharField(db_index=True, max_length=255, unique=True)
    email = models.EmailField(db_index=True, unique=True,  null=True, blank=True)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)

    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']

    objects = UserManager()

    def __str__(self):
        return f"{self.email}"

```

Now what we'll do next is specify to Django to use this new User model as the `AUTH_USER_MODEL`.

```python
# CoreRoot/settings.py
...
AUTH_USER_MODEL = 'core_user.User'
...
```

### Adding User serializer

The next step when working with Django & Django Rest after creating a model is to write a serializer.
Serializer allows us to convert complex Django complex data structures such as `querysets` or model instances in Python native objects that can be easily converted JSON/XML format, but Serializer also serializes JSON/XML to naive Python.

```python
# core/user/serializers.py
from core.user.models import User
from rest_framework import serializers


class UserSerializer(serializers.ModelSerializer):
    created = serializers.DateTimeField(read_only=True)
    updated = serializers.DateTimeField(read_only=True)

    class Meta:
        model = User
        fields = ['public_id', 'username', 'email', 'is_active', 'created', 'updated']
        read_only_field = ['is_active']

```

### Adding User viewset

And the viewset. A viewset is a class-based view, able to handle all of the basic HTTP requests: GET, POST, PUT, DELETE without hard coding any of the logic. And if you have specific needs, you can overwrite those methods.

```python
# core/user/viewsets.py

from core.user.serializers import UserSerializer
from core.user.models import User
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from rest_framework.filters import OrderingFilter


class UserViewSet(viewsets.ModelViewSet):
    http_method_names = ['get']
    serializer_class = UserSerializer
    permission_classes = (IsAuthenticated,)
    filter_backends = [filters.OrderingFilter]
    ordering_fields = ['updated']
    ordering = ['-updated']

    def get_queryset(self):
        if self.request.user.is_superuser:
            return User.objects.all()

    def get_object(self):
        lookup_field_value = self.kwargs[self.lookup_field]

        obj = User.objects.get(lookup_field_value)
        self.check_object_permissions(self.request, obj)

        return obj


```

## Authentication

REST framework provides several authentication schemes out of the box, but we can also implement our custom schemes. We'll use authentication using JWT tokens.
For this purpose, we'll use the `djangorestframework-simplejwt` to implement an access/refresh logic.
Add `rest_framework_simplejwt.authentication.JWTAuthentication` to the list of authentication classes in `settings.py`:

```python
# CoreRoot/settings.py
REST_FRAMEWORK = {
    ...
    'DEFAULT_AUTHENTICATION_CLASSES': (
        ...
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    )
    ...
}
```

The Simple JWT library comes with two useful routes:

- One to obtain access and refresh token (login) 'api/token/'
- And another one to obtain a new access token using the refresh token 'api/token/refresh/'
- It can actually do all the work but there are some issues here :
- The login routes only return a pair of token
- In the user registration flow, the user will be obliged to sign in again to retrieve the pair of tokens.

And since we are using viewsets, there is a problem with consistency.
But here’s the solution :

- Rewrite the login endpoint and serializer to return the pair of tokens and the user object as well
- Generate a pair of tokens when a new user is created and send includes the tokens in the response object
- Make sure that the class-based views will be viewsets.
- Actually, it was a little bit challenging, but shout-out to `djangorestframework-simplejwt` contributors, it’s very simple to read the code, understand how it works and extend it successfully.
- First of all, let's create a package `auth` in `core`.
- In the package, create a file `serializer.py` which will contain the login and register serializers.

```python
# core/auth/serializers.py
from rest_framework import serializers
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.settings import api_settings
from django.contrib.auth.models import update_last_login
from django.core.exceptions import ObjectDoesNotExist
    
from core.user.serializers import UserSerializer
from core.user.models import User
    
    
class LoginSerializer(TokenObtainPairSerializer):

    def validate(self, attrs):
        data = super().validate(attrs)

        refresh = self.get_token(self.user)

        data['user'] = UserSerializer(self.user).data
        data['refresh'] = str(refresh)
        data['access'] = str(refresh.access_token)

        if api_settings.UPDATE_LAST_LOGIN:
            update_last_login(None, self.user)

        return data
    
    
class RegisterSerializer(UserSerializer):
    password = serializers.CharField(max_length=128, min_length=8, write_only=True, required=True)
    email = serializers.EmailField(required=True, write_only=True, max_length=128)

    class Meta:
        model = User
        fields = ['public_id', 'username', 'email', 'password', 'is_active', 'created', 'updated']

    def create(self, validated_data):
        try:
            user = User.objects.get(email=validated_data['email'])
        except ObjectDoesNotExist:
            user = User.objects.create_user(**validated_data)
        return user
```

Then, we can write the viewsets.

```python
# core/auth/viewsets
from rest_framework.response import Response
from rest_framework_simplejwt.views import TokenObtainPairView
from rest_framework.viewsets import ModelViewSet
from rest_framework.permissions import AllowAny
from rest_framework import status
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework_simplejwt.exceptions import TokenError, InvalidToken
from core.auth.serializers import LoginSerializer, RegistrationSerializer
    
    
class LoginViewSet(ModelViewSet, TokenObtainPairView):
    serializer_class = LoginSerializer
    permission_classes = (AllowAny,)
    http_method_names = ['post']

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)

        try:
            serializer.is_valid(raise_exception=True)
        except TokenError as e:
            raise InvalidToken(e.args[0])

        return Response(serializer.validated_data, status=status.HTTP_200_OK)
    
    
class RegistrationViewSet(ModelViewSet, TokenObtainPairView):
    serializer_class = RegisterSerializer
    permission_classes = (AllowAny,)
    http_method_names = ['post']

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)

        serializer.is_valid(raise_exception=True)
        user = serializer.save()
        refresh = RefreshToken.for_user(user)
        res = {
            "refresh": str(refresh),
            "access": str(refresh.access_token),
        }

        return Response({
            "user": serializer.data,
            "refresh": res["refresh"],
            "token": res["access"]
        }, status=status.HTTP_201_CREATED)
    
       
class RefreshViewSet(viewsets.ViewSet, TokenRefreshView):
    permission_classes = (AllowAny,)
    http_method_names = ['post']

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)

        try:
            serializer.is_valid(raise_exception=True)
        except TokenError as e:
            raise InvalidToken(e.args[0])

        return Response(serializer.validated_data, status=status.HTTP_200_OK)
```

The next step is to register the routes.
Create a file `routers.py` in the `core` directory.

```python

# core/routers.py
from rest_framework.routers import SimpleRouter
from core.user.viewsets import UserViewSet
from core.auth.viewsets import LoginViewSet, RegistrationViewSet, RefreshViewSet


routes = SimpleRouter()

# AUTHENTICATION
routes.register(r'auth/login', LoginViewSet, basename='auth-login')
routes.register(r'auth/register', RegistrationViewSet, basename='auth-register')
routes.register(r'auth/refresh', RefreshViewSet, basename='auth-refresh')

# USER
routes.register(r'user', UserViewSet, basename='user')


urlpatterns = [
    *routes.urls
]

```

And the last step, we'll include the routers.urls in the standard list of URL patterns in `CoreRoot`.

```python
# CoreRoot/urls.py
    
from django.contrib import admin
from django.urls import path, include
    
urlpatterns = [
    path('api/', include(('core.routers', 'core'), namespace='core-api')),
]
```

The User endpoints, login, and register viewsets are ready. Don't forget to run migrations and start the server and test the endpoints.

```shell
python manage.py makemigrations
python manage.py migrate
    
python manage.py runserver
```

If everything is working fine, let's create a user with an HTTP Client by requesting `localhost:8000/api/auth/register/`. I'll be using Postman but feel free to use any client.

```json
{
    "email": "testuser@yopmail.com",
    "password": "12345678",
    "username": "testuser"
}
```

## Front-end with React

There are generally two ways to connect Django to your frontend :

- Using Django Rest as a standalone API + React as Standalone SPA. (It needs token-based authentication)
- Or include React in Django templates. (It's possible to use Django built-in authentication features)

The most used pattern is the first one, and we'll focus on it because we have already our token authentication system available.
Make sure you have the latest version of `create-react-app` in your machine.

```shell
yarn create react-app react-auth-app --template typescript
cd react-auth-app
yarn start
```

Then open http://localhost:3000/ to see your app.

But, we'll have a problem. If we try to make a request coming from another domain or origin (here from our frontend with the webpack server), the web browser will throw an error related to the Same Origin Policy. CORS stands for Cross-Origin Resource Sharing and allows your resources to be accessed on other domains.
Cross-Origin Resource Sharing or CORS allows client applications to interface with APIs hosted on different domains by enabling modern web browsers to bypass the Same-origin Policy which is enforced by default.
Let's enable CORS with Django REST by using `django-cors-headers`.

```shell
pip install django-cors-headers
```

If the installation is done, go to your settings.py file and add the package in `INSTALLED_APPS` and the middleware.

```python
INSTALLED_APPS = [
    ...
    'corsheaders',
    ...
]
    
MIDDLEWARE = [
    ...
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    ...
]
```

And add these lines at the end of the `settings.py` file.

```python
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    "http://127.0.0.1:3000"
]
```

We are good now. Let's continue with the front end by adding libraries we'll be using.

### Creating the project

First of all, let's add tailwind and make a basic configuration for the project.

```shell
yarn add tailwindcss@npm:@tailwindcss/postcss7-compat postcss@^7 autoprefixer@^9
```

Since Create React App doesn’t let you override the `PostCSS` configuration natively, we also need to install CRACO to be able to configure Tailwind.

```shell
yarn add @craco/craco
```

Once it's installed, modify these lines in the `package.json` file. Replace `react-`
`scripts` by `craco`.

```json
     "scripts": {
        "start": "craco start",
        "build": "craco build",
        "test": "craco test",
        "eject": "react-scripts eject"
      }
```

Next, we'll create a craco config file in the root of the project, and add `tailwindcss` and `autoprefixer` as plugins.

```json
//craco.config.js
module.exports = {
  style: {
    postcss: {
      plugins: [require("tailwindcss"), require("autoprefixer")],
    },
  },
};
```

Next, we need to create a configuration file for tailwind.
Use `npx tailwindcss-cli@latest init` to generate `tailwind.config.js` file containing the minimal configuration for tailwind.

```json
module.exports = {
  purge: ["./src/**/*.{js,jsx,ts,tsx}", "./public/index.html"],
  darkMode: false, // or 'media' or 'class'
  theme: {
    extend: {},
  },
  variants: {
    extend: {},
  },
  plugins: [],
};
```

The last step will be to include tailwind in the `index.css` file.

```css
/*src/index.css*/

@tailwind base;
@tailwind components;
@tailwind utilities;
```

We are done with the tailwind configuration. 

### Login and Profile Pages

Let's quickly create the Login Page and the Profile Page.

```typescript
// ./src/pages/Login.tsx

import React, { useState } from "react";
import * as Yup from "yup";
import { useFormik } from "formik";
import { useDispatch } from "react-redux";
import axios from "axios";
import { useHistory } from "react-router";

function Login() {
  const [message, setMessage] = useState("");
  const [loading, setLoading] = useState(false);
  const dispatch = useDispatch();
  const history = useHistory();

  const handleLogin = (email: string, password: string) => {
    //
  };

  const formik = useFormik({
    initialValues: {
      email: "",
      password: "",
    },
    onSubmit: (values) => {
      setLoading(true);
      handleLogin(values.email, values.password);
    },
    validationSchema: Yup.object({
      email: Yup.string().trim().required("Le nom d'utilisateur est requis"),
      password: Yup.string().trim().required("Le mot de passe est requis"),
    }),
  });

  return (
    <div className="h-screen flex bg-gray-bg1">
      <div className="w-full max-w-md m-auto bg-white rounded-lg border border-primaryBorder shadow-default py-10 px-16">
        <h1 className="text-2xl font-medium text-primary mt-4 mb-12 text-center">
          Log in to your account 🔐
        </h1>
        <form onSubmit={formik.handleSubmit}>
          <div className="space-y-4">
            <input
              className="border-b border-gray-300 w-full px-2 h-8 rounded focus:border-blue-500"
              id="email"
              type="email"
              placeholder="Email"
              name="email"
              value={formik.values.email}
              onChange={formik.handleChange}
              onBlur={formik.handleBlur}
            />
            {formik.errors.email ? <div>{formik.errors.email} </div> : null}
            <input
              className="border-b border-gray-300 w-full px-2 h-8 rounded focus:border-blue-500"
              id="password"
              type="password"
              placeholder="Password"
              name="password"
              value={formik.values.password}
              onChange={formik.handleChange}
              onBlur={formik.handleBlur}
            />
            {formik.errors.password ? (
              <div>{formik.errors.password} </div>
            ) : null}
          </div>
          <div className="text-danger text-center my-2" hidden={false}>
            {message}
          </div>

          <div className="flex justify-center items-center mt-6">
            <button
              type="submit"
              disabled={loading}
              className="rounded border-gray-300 p-2 w-32 bg-blue-700 text-white"
            >
              Login
            </button>
          </div>
        </form>
      </div>
    </div>
  );
}

export default Login;
```

Here's a preview :

![Preview of Login Page](https://cdn.hashnode.com/res/hashnode/image/upload/v1624673265869/pv-SIiuEw.png)

And the profile page :

```typescript
// ./src/pages/Profile.tsx
    
import React from "react";
import { useDispatch } from "react-redux";
import { useHistory } from "react-router";

const Profile = () => {
  const dispatch = useDispatch();
  const history = useHistory();

  const handleLogout = () => {
    //
  };
  return (
    <div className="w-full h-screen">
      <div className="w-full p-6">
        <button
          onClick={handleLogout}
          className="rounded p-2 w-32 bg-red-700 text-white"
        >
          Deconnexion
        </button>
      </div>
      <div className="w-full h-full text-center items-center">
        <p className="self-center my-auto">Welcome</p>
      </div>
    </div>
  );
};

export default Profile;
```

And here's the preview :

![Screenshot 2021-06-26 at 02-10-29 React App.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1624673462400/lg0i8RpmJ.png)

### Env variables configurations

And the final step, we'll be making requests on an API. It's a good practice to configure environment variables. Fortunately, React allows us to make basic environment configurations.
Create a `.env` file at the root of the project and put this here.

```shell
./.env
REACT_APP_API_URL=localhost:8000/api
```

### Add Redux Store

Redux is a library to manage the global state in our application.
Here, we want the user to log in and go to the Profile Page. It will only work if the login is correct.
But that's not all: if the user has no active session -meaning that the refresh is expired or there is no trace of this user account or tokens in the storage of the frontend -  he is directly redirected to the login page.

To make things simple, here's what we're going to do:

- create a persisted store with (redux-persist) for our project, and write actions using slices from `redux-toolkit` to save, account state, and tokens when the user signs in. We'll also write an action for logout.
- create a Protected route component, that'll check if the state of the user account null or exists and then redirect the user according to the results.

First of all, let's add the dependencies we need to configure the store.

```shell
yarn add @reduxjs/toolkit redux react-redux redux-persist
```

Then, create a folder named `store` in `src`.
Add in this directory another folder named `slices` and create in this directory a file named `auth.ts`.
With Redux, a slice is a collection of reducer logic and actions for a single feature of our app. 
But before adding content to this file, we need to write the interface for the user account.

```typescript
// ./src/types.ts

export interface AccountResponse {
  user: {
    id: string;
    email: string;
    username: string;
    is_active: boolean;
    created: Date;
    updated: Date;
  };
  access: string;
  refresh: string;
}
```

And now, we can write the authentication slice `authSlice`.

```typescript
// ./src/store/slices/auth.ts

import { createSlice, PayloadAction } from "@reduxjs/toolkit";
import { AccountResponse } from "../../types";

type State = {
  token: string | null;
  refreshToken: string | null;
  account: AccountResponse | null;
};

const initialState: State = { token: null, refreshToken: null, account: null };

const authSlice = createSlice({
  name: "auth",
  initialState,
  reducers: {
    setAuthTokens(
      state: State,
      action: PayloadAction<{ token: string; refreshToken: string }>
    ) {
      state.refreshToken = action.payload.refreshToken;
      state.token = action.payload.token;
    },
    setAccount(state: State, action: PayloadAction<AccountResponse>) {
      state.account = action.payload;
    },
    logout(state: State) {
      state.account = null;
      state.refreshToken = null;
      state.token = null;
    },
  },
});

export default authSlice;
```

Now, move inside the store directory and create a file named `index.ts`. And add the following content.

```typescript
// ./src/store/index.ts

import { configureStore, getDefaultMiddleware } from "@reduxjs/toolkit";
import { combineReducers } from "redux";
import {
  FLUSH,
  PAUSE,
  PERSIST,
  persistReducer,
  persistStore,
  PURGE,
  REGISTER,
  REHYDRATE,
} from "redux-persist";
import storage from "redux-persist/lib/storage";
import authSlice from "./slices/auth";

const rootReducer = combineReducers({
  auth: authSlice.reducer,
});

const persistedReducer = persistReducer(
  {
    key: "root",
    version: 1,
    storage: storage,
  },
  rootReducer
);

const store = configureStore({
  reducer: persistedReducer,
  middleware: getDefaultMiddleware({
    serializableCheck: {
      ignoredActions: [FLUSH, REHYDRATE, PAUSE, PERSIST, PURGE, REGISTER],
    },
  }),
});

export const persistor = persistStore(store);
export type RootState = ReturnType<typeof rootReducer>;

export default store;
```

Now the store has been created, we need to make the `store` accessible for all components by wrapping `<App />` (top-level-component) in :

```typescript
// ./src/App.tsx
    
import React from "react";
import { BrowserRouter as Router, Switch, Route } from "react-router-dom";
import { Login, Profile } from "./pages";
import store, { persistor } from "./store";
import { PersistGate } from "redux-persist/integration/react";
import { Provider } from "react-redux";
import ProtectedRoute from "./routes/ProtectedRoute";

export default function App() {
  return (
    <Provider store={store}>
      <PersistGate persistor={persistor} loading={null}>
        <Router>
          <div>
            <Switch>
              <Route exact path="/login" component={Login} />
              <ProtectedRoute exact path="/" component={Profile} />
            </Switch>
          </div>
        </Router>
      </PersistGate>
    </Provider>
  );
}
```

The store is accessible by all components in our application now. The next step is to build a `<ProtectedRoute />` component to help us hide pages that require sessions from the other ones.

### Adding routes

We'll build the `<ProtectedRoute />`component using React Router. 
React Router is a standard library for routing in React. It enables the navigation among views of various components in a React Application, allows changing the browser URL, and keeps the UI in sync with the URL.
In our application, If the user tries to access a protected page, we'll be redirected to the Login Page.

```shell
cd src & mkdir routes
cd routes
```

In the routes, directory creates a file named `ProtectedRoute.tsx` , and write this :

```typescript
// ./src/routes/ProtectedRoute.tsx
    
import React from "react";
import { Redirect, Route, RouteProps } from "react-router";
import { useSelector } from "react-redux";
import { RootState } from "../store";

const ProtectedRoute = (props: RouteProps) => {
  const auth = useSelector((state: RootState) => state.auth);

  if (auth.account) {
    if (props.path === "/login") {
      return <Redirect to={"/"} />;
    }
    return <Route {...props} />;
  } else if (!auth.account) {
    return <Redirect to={"/login"} />;
  } else {
    return <div>Not found</div>;
  }
};

export default ProtectedRoute;
```

The first step here is to get the global state of `auth`. Actually, every time a user successfully signs in, we'll use the slices to persist the account state and the tokens in the storage.
If there is an account object, that means that there is an active session.
Then, we use this state to check if we have to redirect the user to the protected page `return <Route {...props} />;` or he is directly redirected to the login page `return <Redirect to={"/login"} />;`.
The last and final step is to rewrite the Login and Profile Page. Let's start with the Login Page.

```typescript
// ./src/pages/Login.tsx
import authSlice from "../store/slices/auth";
  
    ...
    const handleLogin = (email: string, password: string) => {
        axios
          .post(`${process.env.REACT_APP_API_URL}/auth/login/`, { email, password })
          .then((res) => {
            dispatch(
              authSlice.actions.setAuthTokens({
                token: res.data.access,
                refreshToken: res.data.refresh,
              })
            );
            dispatch(authSlice.actions.setAccount(res.data.user));
            setLoading(false);
            history.push("/");
          })
          .catch((err) => {
            setMessage(err.response.data.detail.toString());
          });
      };
    ...
```

And the profile Page,

```typescript
// ./src/pages/Profile.tsx

import authSlice from "../store/slices/auth";

    ...
    const handleLogout = () => {
        dispatch(authSlice.actions.logout());
        history.push("/login");
      };
    ...
```

And we're done with the front end. Start your server again and try to log in with the user-created with POSTMAN.
That's some basic stuff if you need to build an authentication system with React and Django.
However, the application has some issues, and trying to perfect it here was only going to increase the length of the article.
So here are the issues and the solutions :

- **JWT** : JSON Web Tokens come with some issues you should be aware of if you to make a great usage. Feel free to check this [article](https://curity.io/resources/learn/jwt-best-practices/), to learn how to use JWT effectively.
- **PostgreSQL** : For this tutorial, I used sqlite3 to make things faster. If you are going to a production or a staging server, always use a database motor with good performances.
- **A refresh client** : Actually, the user is logged, but when the time will come to make a request, you'll have only 5 minutes of access to the content.
- The **access token** is used to access protected resources from the server then every 5 minutes, you should get a new one from the server using a refresh token. Then, you'll need to write an async refresh logic with `axios` and fortunately, you can read this article to better understand how you can do it. ;)

## Conclusion

In this article, We learned to build a CRUD application web with Django and React. And as every article can be made better so your suggestion or questions are welcome in the comment section. 😉

Check the code of the Django app [here](https://github.com/koladev32/django-auth-react-tutorial) and the React App [here](https://github.com/koladev32/django-react-auth-app).
