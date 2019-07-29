# GeoPins

\$ npm run dev

## Part 1: Prepare

### Section 2: Building our GraphQL Server

**4. Creating our GraphQL Server**

server.js

```javascript
const { ApolloServer } = require("apollo-server");

const typeDefs = require("./typeDefs");
const resolvers = require("./resolvers");

const server = new ApolloServer({
  typeDefs: typeDefs,
  resolvers: resolvers
});

server.listen().then(({ url }) => console.log(`Server listening on ${url}`));
```

typeDefs.js

```javascript
const { gql } = require("apollo-server");

module.exports = gql`
  type User {
    _id: ID
    name: String
    email: String
    picture: String
  }

  type Query {
    me: User
  }
`;
```

resolvers.js

```javascript
const user = {
  // Create a fake data
  _id: "1",
  name: "Zach",
  email: "zachzwy@gmail.com",
  picture: "https://cloudinary.com/asdf"
};

module.exports = {
  Query: {
    me: () => user
  }
};
```

**5. Creating Database with MongoDB Atlas**

Create an account on MongoDB

Create a new file called .env

.env

```javascript
MONGO_URI=mongodb+srv://wenyuzhang:wenyuzhang@geopin-ohtej.mongodb.net/test?retryWrites=true&w=majority
```

server.js

```javascript
// ...

const mongoose = require("mongoose");
require("dotenv").config();

// ...

mongoose
  .connect(process.env.MONGO_URI, {
    useNewUrlParser: true
  })
  .then(() => console.log("DB connected!"))
  .catch(e => console.error(e));

// ...
```

**6. Creating Mongoose Models for User / Pin Data**

Now that we've connected our server to our MongoD.B. Atlas
database. Now we need to tell our database about the different
types of data that we have in our app, just like we did in the
typeDefs.js

Under **models** folder, create a new file called User.js,
inside this new file, create a UserSchema

User.js

```javascript
const mongoose = require("mongoose");

const UserSchema = new mongoose.Schema({
  name: String,
  email: String,
  picture: String
});
// We can omit _id field because it is generated by
// our database whenever we generate a new user entry

module.exports = mongoose.model("User", UserSchema);
```

Add new Pin data in typeDefs.js

typeDefs.js

```javascript
// ...

type Pin {
  _id: ID
  createdAt: String
  title: String
  content: String
  image: String
  latitute: Float
  longitute: Float
  author: User
  comments: [Comment]
}

type Comment {
  text: String
  createdAt: String
  author: User
}

// ...
```

Under **models** folder, create a new file called Pin.js,
inside this new file, create a PinSchema

Pin.js

```javascript
const mongoose = require("mongoose");

const PinSchema = new mongoose.Schema(
  {
    title: String,
    content: String,
    image: String,
    latitude: Number,
    longitude: Number,
    author: { type: mongoose.Schema.ObjectId, ref: "User" },
    comments: [
      {
        text: String,
        createdAt: { type: Date, default: Date.now },
        author: { type: mongoose.Schema.ObjectId, ref: "User" }
      }
    ]
  },
  { timestamps: true }
);

module.exports = mongoose.model("Pin", PinSchema);
```

### Section 3: Social Login with Google OAuth 2.0

**7. Exploring our React App**

When you're inside client folder, run

\$ npm run dev

**8. Setting up Google OAuth**

Login to concole.developer.google.com. Create a new project, enable Google+ API.
Create credentials.

Add client ID to the .env file

.env

```javascript
//...

OAUTH_CLIENT_ID =
  29144684875 - ofvjb22rtvd1t90bn0db34d91r4l2e84.apps.googleusercontent.com;
```

**9. Adding Google Login Button**

src/pages/Splash.js

```javascript
import React from "react";

import Login from "../components/Auth/Login";

const Splash = () => {
  return <Login />;
};

export default Splash;
```

src/components/Auth/Login.js

```javascript
// ...

import { GoogleLogin } from "react-google-login";

// ...

const Login = ({ classes }) => {
  const onSuccess = googleUser => {
    const id_token = googleUser.getAuthResponse().id_token;
    console.log(id_token);
  };

  const onFailure = () => console.log("Login failed");

  return (
    <GoogleLogin
      clientId="29144684875-ofvjb22rtvd1t90bn0db34d91r4l2e84.apps.googleusercontent.com"
      onSuccess={onSuccess}
      onFailure={onFailure}
      isSignedIn={true}
    />
  );
};

// ...
```

**10. Authenticating Users from Apollo Server**

It means sending over this ID token to our server checking
to see if it's valid, getting the user's Google information there
and then when we execute a query from the client asking for the
current user's information we'll send it back from the server
upon which we'll store that information in our app and redirect them
to the app component.

src/components/Auth/Login.js

```javascript
// ...

import { GraphQLClient } from "graphql-request";

// ...

const ME_QUERY = `
{
  me {
		_id,
    name,
    email,
    picture
  }
}
`;

// ...

const Login = ({ classes }) => {
  const onSuccess = async googleUser => {
    // ...

    const client = new GraphQLClient("http://localhost:4000/graphql", {
      headers: { authorization: id_token }
    });
    const data = await client.request(ME_QUERY);
    console.log({ data });
  };

  // ...
};
```

server.js

```javascript
const { findOrCreateUser } = require("./controllers/userController");

const server = new ApolloServer({
  // ...

  context: async ({ req }) => {
    let authToken = null;
    let currentUser = null;
    try {
      authToken = req.headers.authorization;
      if (authToken) {
        // find or create user
        currentUser = await findOrCreateUser(authToken);
      }
    } catch (err) {
      console.error(`Unable to authenticate user with token ${authToken}`);
    }
    return { currentUser };
  }
});
```

Create a new file userController.js under the folder controllers

userController.js

```javascript
const User = require("../models/User");
const { OAuth2Client } = require("google-auth-library");

const client = new OAuth2Client(process.env.OAUTH_CLIENT_ID);

exports.findOrCreateUser = async token => {
  // verify auth token
  const googleUser = await verifyAuthToken(token);
  // check if the user exists
  const user = await checkIfUserExists(googleUser.email);
  // if user exists,return them; otherwise, create a new user
  return user ? user : createNewUser(googleUser);
};

const verifyAuthToken = async token => {
  try {
    const ticket = await client.verifyIdToken({
      idToken: token,
      audience: process.env.OAUTH_CLIENT_ID
    });
    return ticket.getPayload();
  } catch (err) {
    console.error("Error verifying auth token", err);
  }
};

const checkIfUserExists = async email => await User.findOne({ email }).exec();

const createNewUser = googleUser => {
  const { name, email, picture } = googleUser;
  const user = { name, email, picture };
  return new User(user).save(); // Add a new User to database
};
```

resolvers.js

```javascript
const { AuthenticationError } = require("apollo-server");

// ...

const authenticated = next => (root, args, ctx, info) => {
  if (!ctx.currentUser) {
    throw new AuthenticationError("You must be logged in.");
  }
  return next(root, args, ctx, info);
};

module.exports = {
  Query: {
    me: authenticated((root, args, ctx) => ctx.currentUser)
  }
};
```

### Section 4: Managing App State with useReducer / userContext Hooks

**11. Managing App State with useContext / useReducer**

Create a new file under client/src, called context.js

client/src/context.js

```javascript
import { createContext } from "react";

const Context = createContext({
  currentUser: null
});

export default Context;
```

client/src/index.js

```javascript
import React, { useContext, useReducer } from "react";

// ...

import Context from "./context";
import reducer from "./reducer";

// ...

const Root = () => {
  const initialState = useContext(Context);
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <Router>
      <Context.Provider value={{ state, dispatch }}>
        <Switch>
          <Route exact path="/" component={App} />
          <Route path="/login" component={Splash} />
        </Switch>
      </Context.Provider>
    </Router>
  );
};
```

Create a new file called reducer.js under client/src

client/src/reducer.js

```javascript
const reducer = (state, action) => {
  switch (action.type) {
    case "LOGIN_USER":
      return {
        ...state,
        currentUser: action.payload
      };
    default:
      return state;
  }
};

export default reducer;
```

client/src/component/Auth/Login.js

```javascript
import React, { useContext } from "react";
import Context from "../../context";
// ...

const Login = ({ classes }) => {
  const { dispatch } = useContext(Context);

  const onSuccess = async googleUser => {
    // ...
    dispatch({
      type: "LOGIN_USER",
      payload: data.me
    });
  };

  // ...
};
```

**12. Styling Splash Page / App Cleanup**

Login.js

```javascript
const Login = ({ classes }) => {

  // ...

  return (
    <div className={classes.root}>
      <Typography
        component='h1'
        variant='h3'
        gutterBottom
        noWrap
        style={{ color: 'rgb(66, 133, 244)' }}
      >
        Welcome
      </Typography>

      {//...}

    </div>
  );
};
```

Move ME_QUERY to src/graphql/queries.js

### Section 5: Protecting Our App Route

**13. Creating Protected Route for App**

reducer.js

```javascript
const reducer = (state, action) => {
  switch (action.type) {
    // ...

    case "IS_LOGGED_IN":
      return {
        ...state,
        isAuth: action.payload
      };

    // ...
  }
};
```

context.js

```javascript
// ...
const Context = createContext({
  // ...
  isAuth: false
});
// ...
```

Login.js

```javascript
const Login = ({ classes }) => {
  // ....
  const onSuccess = async googleUser => {
    // ...
    dispatch({
      type: "IS_LOGGED_IN",
      payload: googleUser.isSignedIn
    });
  };
  // ...
};
```

Splash.js

```javascript
// ...
import { Redirect } from "react-router-dom";
import Context from "../context";

const Splash = () => {
  const { state } = useContext(Context);
  // If user is logged in, redirect to app page
  return state.isAuth ? <Redirect to="/" /> : <Login />;
};

// ...
```

Create a new file called ProtectedRoute.js under src folder

ProtectedRoute.js

```javascript
import React, { useContext } from "react";
import { Route } from "react-router-dom";
import Context from "./context";
import Redirect from "react-router-dom/Redirect";

const ProtectedRoute = ({ component: Component, ...rest }) => {
  const { state } = useContext(Context);
  return (
    <Route
      render={props =>
        !state.isAuth ? <Redirect to="/login" /> : <Component {...props} />
      }
      {...rest}
    />
  );
};

export default ProtectedRoute;
```

index.js

```javascript
const Root = () => {
  // ...
  return (
    <Router>
      <Context.Provider value={{ state, dispatch }}>
        <Switch>
          <ProtectedRoute exact path="/" component={App} />
          <Route path="/login" component={Splash} />
        </Switch>
      </Context.Provider>
    </Router>
  );
};
```

## Part 2: Feature

### Section 6: Building the Header

**14. Building Header Component**

Change the text on login button

components/Auth/Login.js

```javascript
// ...
<GoogleLogin
  {/* ... */}
  buttonText="Login with Google"
  {/* ... */}
/>
// ...
```

src/components/Header.js

```javascript
// ...
import Context from "../context.js";
// ...
const Header = ({ classes }) => {
  // useContext accepts a context object (the value returned from React.createContext)
  // and returns the current context value for that context.
  // The current context value is determined by the value prop
  // of the nearest <MyContext.Provider> above the calling component in the tree.
  const { state } = useContext(Context);
  const { currentUser } = state;

  return (
    <div className={classes.root}>
      <AppBar position="static">
        <Toolbar>
          {/* Title / Logo */}
          <div className={classes.grow}>
            <MapIcon className={classes.icon} />
            <Typography component="h1" variant="h6" color="inherit" noWrap>
              GeoPins
            </Typography>
          </div>
          {/* Current user info */}
          <div>
            {currentUser && (
              <div className={classes.grow}>
                <img
                  className={classes.picture}
                  src={currentUser.picture}
                  alt={currentUser.name}
                />
                <Typography variant="h5" color="inherit" noWrap>
                  {currentUser.name}
                </Typography>
              </div>
            )}
          </div>
          {/* Sign out button */}
          <div />
        </Toolbar>
      </AppBar>
    </div>
  );
};
// ...
```

src/pages/App.js

```javascript
import Header from "../components/Header";

const App = () => {
  return <Header />;
};
```

**15. Build Signout Button**

src/components/Header.js

```javascript
// ...
import Signout from "./Auth/Signout";
// ...
<AppBar>
  <ToolBar>
    {/* ... */}
    <Signout />
    {/* ... */}
  </ToolBar>
</AppBar>;
// ...
```

src/components/Auth/Signout.js

```javascript
// ...
import React, { useContext } from "react";
import { GoogleLogout } from "react-google-login";
import Context from "../../context";

const Signout = ({ classes }) => {
  // Import Context so to use dispatch function
  const { dispatch } = useContext(Context);

  // When user click signout button, we dispath an action
  // that's going to clear from our global state
  const onSignout = () => {
    dispatch({ type: "SIGNOUT_USER" });
  };

  return (
    <GoogleLogout
      // onLogoutSuccess function will evoke when we click signout
      onLogoutSuccess={onSignout}
      // Make our own custom button, so we use render prop here.
      // One thing about custom button is that we need to
      // pass onClick prop manually to make logout work
      render={({ onClick }) => (
        <span className={classes.root} onClick={onClick}>
          <Typography variant="body1" className={classes.buttonText}>
            Signout
          </Typography>
          <ExitToAppIcon className={classes.buttonIcon} />
        </span>
      )}
    />
  );
};
```

Add "SIGNOUT_USER" action handler in our reducer function

src/reducer.js

```javascript
// ...
case "SIGNOUT_USER":
  return {
    ...state,
    isAuth: false,
    currentUser: null
  };
// ...
```

### Section 7: Building the Map / User Geolocation

**16. Creating and Styling our Map**

src/pages/App.js

```javascript
import Header from "../components/Header";

const App = () => {
  return (
    <>
      <Header />
      <Map />
    </>
  );
};
```

src/components/Map.js

```javascript
const Map = ({ classes }) => {
  const INITIAL_VIEWPORT = {
    latitude: 37.76,
    longitude: -122.4376,
    zoom: 13
  };

  const [viewport, setViewport] = useState(INITIAL_VIEWPORT);

  return (
    <div className={classes.root}>
      <ReactMapGL
        width="100vw"
        height="calc(100vh - 64px)"
        mapStyle="mapbox://styles/mapbox/streets-v9"
        mapboxApiAccessToken="pk.eyJ1IjoiemFjaHp3eSIsImEiOiJjanczeWZ1aGYxOW05M3pwczRkZ3A1NGJ4In0.BFX8cW_ZygtvgjIvrwhT1g"
        onViewportChange={newViewport => setViewport(newViewport)}
        {...viewport}
      >
        {/* Navigation control */}
        <div className={classes.navigationControl}>
          <NavigationControl
            onViewportChange={newViewport => setViewport(newViewport)}
          />
        </div>
      </ReactMapGL>
    </div>
  );
};
```

**17. Placing a Pin at User's Current Position**

src/components/Map.js

```javascript
const Map = ({ classes }) => {
  // ...
  const [userLocation, setUserLocation] = useState(null);

  const getUserLocation = () => {
    if ("geolocation" in navigator) {
      navigator.geolocation.getCurrentPosition(position => {
        const { latitude, longitude } = position.coords;
        setViewport({ ...viewport, latitude, longitude });
        setUserLocation({ latitude, longitude });
      });
    }
  };

  useEffect(() => getUserLocation(), []);

  return (
    <div className={classes.root}>
      <ReactMapGL
      // ...
      >
        {/* ... */}

        {/* Pin icon to display user's current location */}
        {userLocation && (
          <Marker
            latitude={userLocation.latitude}
            longitude={userLocation.longitude}
            offsetLeft={-19}
            offsetTop={-37}
          >
            <PinIcon size={40} color="red" />
          </Marker>
        )}
      </ReactMapGL>
    </div>
  );
};
```

src/components/PinIcon.js

```javascript
import React from "react";
import PlaceTwoTone from "@material-ui/icons/PlaceTwoTone";

export default ({ size, color, onClick }) => (
  <PlaceTwoTone onClick={onClick} style={{ fontSize: size, color }} />
);
```

### Section 8: Creating Blog Area / Adding Draft Pins

**18. Adding Draft Pin**

1. Add onClick prop on <ReactMapGL /> and handleClick function on <Map />

src/components/Map.js

```javascript
// ...
const handleClick = ({ lngLat, leftButton }) => {
  if (!leftButton) return;
};
// ...
<ReactMapGL
  // ...
  onClick={handleClick}
>
```

2. Add draft state to global state (context)

src/context.js

```javascript
const Context = createContext({
  // ...
  draft: null
});
```

3. Bring context to Map component

src/components/Map.js

```javascript
// ...
import Context from "../context";
// ...
```

4. CREATE_DRAFT and UPDATE_DRAFT_LOCATION reducer

src/reducer.js

```javascript
case "CREATE_DRAFT":
  return {
    ...state,
    draft: {
      latitude: 0,
      longitude: 0
    }
  };
case "UPDATE_DRAFT_LOCATION":
  return {
    ...state,
    draft: action.payload
  };
```

5. Update handleClick function add draft Marker on Map

src/components/Map.js

```javascript
const handleClick = ({ lngLat, leftButton }) => {
  if (!leftButton) return;
  if (!state.draft) {
    dispatch({ type: "CREATE_DRAFT" });
  }
  dispatch({
    type: "UPDATE_DRAFT_LOCATION",
    payload: {
      latitude: lngLat[1],
      longitude: lngLat[0]
    }
  });
};

// ...

{
  /* Draft pin */
}
{
  state.draft && (
    <Marker
      latitude={state.draft.latitude}
      longitude={state.draft.longitude}
      offsetLeft={-19}
      offsetTop={-37}
    >
      <PinIcon size={40} color="hotpink" />
    </Marker>
  );
}
```

**19. Adding Blog Area for Pin Content**

1. Add Blog component to Map component under ReactMapGl component

src/components/Map.js

```javascript
// ...
{
  /* Blog area */
}
<Blog />;
// ...
```

2. Import Context and Set up NoContent, CreatePin, PinContent component inside Blog component

src/components/Blog.js

```javascript
const Blog = ({ classes }) => {
  const { state } = useContext(Context);
  const { draft } = state;

  let BlogContext;
  if (!draft) {
    BlogContext = NoContent;
  } else {
    BlogContext = CreatePin;
  }
  return (
    <Paper className={classes.root}>
      <BlogContext />
    </Paper>
  );
};
```

**20. Building / Styling Blog Components**

1. Styling NoContent component

src/components/Pin/NoContent.js

```javascript
// ...

const NoContent = ({ classes }) => (
  <div className={classes.root}>
    <ExploreIcon className={classes.icon} />
    <Typography
      noWrap
      component="h2"
      variant="h6"
      align="center"
      color="textPrimary"
      gutterBottom
    >
      Click on the map to add a pin
    </Typography>
  </div>
);

// ...
```

2. Editing CreatePin component

src/components/Pin/CreatePin.js

```javascript
// ...

const CreatePin = ({ classes }) => (
  <form className={classes.form}>
    <Typography
      className={classes.alignCenter}
      component="h2"
      variant="h4"
      color="secondary"
    >
      <LandscapeIcon className={classes.iconLarge} /> Pin Location
    </Typography>
    <div>
      <TextField name="title" label="Title" placeholder="Insert pin title" />
      <input
        accept="image/*"
        id="image"
        type="file"
        className={classes.input}
      />
      <label htmlFor="image">
        <Button component="span" size="small" className={classes.button}>
          <AddAPhotoIcon />
        </Button>
      </label>
    </div>
    <div className={classes.contentField}>
      <TextField
        name="content"
        label="Content"
        multiline
        rows="6"
        margin="normal"
        fullWidth
        variant="outlined"
      />
    </div>
    <div>
      <Button className={classes.button} variant="contained" color="primary">
        <ClearIcon className={classes.leftIcon} />
        Discard
      </Button>
      <Button
        type="submit"
        className={classes.button}
        variant="contained"
        color="secondary"
      >
        Submit
        <SaveIcon className={classes.rightIcon} />
      </Button>
    </div>
  </form>
);

// ...
```

**21. Managing Pin Content State and Deleting Draft Pins**

1. Add three states: title, image, content

src/components/Pin/CreatePin.js

```javascript
// ...

const [title, setTitle] = useState("");
const [image, setImage] = useState("");
const [content, setContent] = useState("");

const handleSubmit = e => {
  e.preventDefault();
  console.log({ title, image, content });
};

// ...

<TextField
  // ...
  onChange={e => setTitle(e.target.value)}
/>

// ...

<input
  // ...
  onChange={e => setImage(e.target.files[0])}
/>

// ...

<TextField
  // ...
  onChange={e => setContent(e.target.value)}
/>

// ...

<Button
  // ...
  disabled={!title.trim() || !image || !content.trim()}
  onClick={handleSubmit}
/>

// ...

```

2. Allow user to discard and submit pin

src/components/Pin/CreatePin.js

```javascript
// ...

import Content from "../../context";

// ...

const { dispatch } = useContext(Content);

// ...

const handleDeleteDraft = () => {
  setTitle("");
  setImage("");
  setContent("");
  dispatch({ type: "DELETE_DRAFT" });
};

// ...

<Button
  // ...
  onClick={handleDeleteDraft}
/>;

// ...
```

### Section 9: Image Uploads with Cloudinary Web API

**22. Uploading Images with Cloudinary**

1. Signup an free account on Cloudinary, use familiar cloud name, add unsigned upload

2. handleImageUpload

src/components/Pin/CreatePin.js

```javascript
// ...

import axios from "axios";

// ...

const handleImageUpload = async () => {
  const data = new FormData();
  data.append("file", image);
  data.append("upload_preset", "geopins");
  data.append("cloud_name", "wenyuzhang");
  const res = await axios.post(
    "https://api.cloudinary.com/v1_1/wenyuzhang/image/upload",
    data
  );
  return res.data.url;
};
```

## Part 3: More Feature

### Section 10: Creating New User Pins

**23. Creating New Pins with CREATE_PIN Mutation**

1. Add Mutation in typeDefs.js

typeDefs.js

```javascript
module.exports = gql`
  input CreatePinInput {
    title: String
    image: String
    content: String
    latitude: Float
    longitude: Float
  }

  type Mutation {
    createPin(input: CreatePinInput!): Pin
  }
`;
```

2. Update resolvers

resolvers.js

```javascript
module.exports = {
  Query: {
    me: authenticated((root, args, ctx) => ctx.currentUser)
  },
  Mutation: {
    createPin: authenticated(async (root, args, ctx) => {
      const newPin = await new Pin({
        ...args.input,
        author: ctx.currentUser._id
      }).save();
      const pinAdded = await Pin.populate(newPin, "author");
      return pinAdded;
    })
  }
};
```

3. src/graphql/mutation.js

```javascript
export const CREATE_PIN_MUTATION = `
  mutation($title: String!, $image: String!, $content: String!, $latitude: Float!, $longitude: Float!) {
    createPin(input: {
      title: $title,
      image: $image,
      content: $content,
      latitude: $latitude,
      longitude: $longitude
    }) {
      _id
      createdAt
      title
      image
      content
      latitude
      longitude
      author {
        _id
        name
        email
        picture
      }
    }
  }
`;
```

4. CreatePin.js

src/components/Pin/CreatePin.js

```javascript
const handleSubmit = async e => {
  try {
    e.preventDefault();
    setSubmitting(true);
    const idToken = window.gapi.auth2
      .getAuthInstance()
      .currentUser.get()
      .getAuthResponse().id_token;
    const client = new GraphQLClient("http://localhost:4000/graphql", {
      headers: { authorization: idToken }
    });
    const url = await handleImageUpload();
    const { latitude, longitude } = state.draft;
    const variables = {
      title,
      image: url,
      content,
      latitude,
      longitude
    };
    const { createPin } = await client.request(CREATE_PIN_MUTATION, variables);
    handleDeleteDraft();
    console.log("Pin created ", { createPin });
  } catch (err) {
    setSubmitting(false);
    console.error("Error createing pin ", err);
  }
};
```

### Section 11: Making Costom useClient Hook

**24. Create Costom GraphQL Request Hook**

1. Create a new file

src/client.js

```javascript
import { useState, useEffect } from "react";
import { GraphQLClient } from "graphql-request";

export const BASE_URL =
  process.env.NODE_ENV === "production"
    ? "<insert-production-url>"
    : "http://localhost:4000/graphql";

export const useClient = () => {
  const [idToken, setIdToken] = useState("");

  useEffect(() => {
    const token = window.gapi.auth2
      .getAuthInstance()
      .currentUser.get()
      .getAuthResponse().id_token;
    setIdToken(token);
  }, []);

  return new GraphQLClient(BASE_URL, {
    headers: { authorization: idToken }
  });
};
```

2. Also update CreatePin.js and Login.js

### Section 12: Getting / Displaying Created Pins

**25. Displaying Created Pins on the Map**

### Section 13: Popups and Highlighting New Pins

**26. Highlighting Newly Created Pins**
**27. Adding Popup to our Pins**

### Section 14: Deleting User Pins

**28. Deleting Pins with DELETE_PIN Mutation**

### Section 15: Displaying Pin Content

**29. Building Out / Styling Pin Content**

### Section 16: Add Comment Function ality

**30. Building out Components to Create / Display User Comments**
**31. Creating Comments with CREATE_COMMENT_MUTATION**

## Part 4: Subscription

### Section 17: Client Error Handling

**32. Hanlding Expired Auth Token Errors**

### Section 18: Live Data with GraphQL Subscriptions / Apollo Client

**33. Setting up Subscriptions on the Backend**
**34. Subscribing to Live Data Changes with Apollo Client**

## Part 5: Styling

### Section 19: Styling our App for Mobile / useMediaQuery

**35. useMediaQuery for Easy Mobile / Response Design**

### Section 20: Improving our App / Fixing App Issues

**36. Fixing App Issues**

### Section 21: Deploying our App

**37. Deploying with Now v2 and Heroku**

### Section 22: BONUS

**38. Bonus Lecture**
