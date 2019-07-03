# GeoPins

$ npm run dev

## Part 1: Prepare

### Section 2: Building our GraphQL Server

**4. Creating our GraphQL Server**

server.js
```javascript
const { ApolloServer } = require('apollo-server');

const typeDefs = require('./typeDefs');
const resolvers = require('./resolvers');

const server = new ApolloServer({
  typeDefs: typeDefs,
  resolvers: resolvers
});

server.listen().then(({ url }) => console.log(`Server listening on ${url}`));
```

typeDefs.js
```javascript
const { gql } = require('apollo-server');

module.exports = gql`
  type User {
    _id: ID,
    name: String,
    email: String,
    picture: String
  }

  type Query {
    me: User
  }
`
```

resolvers.js
```javascript
const user = { // Create a fake data
  _id: "1",
  name: "Zach",
  email: "zachzwy@gmail.com",
  picture: "https://cloudinary.com/asdf"
}

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

const mongoose = require('mongoose');
require('dotenv').config();

// ...

mongoose
  .connect(process.env.MONGO_URI, {
  useNewUrlParser: true
})
  .then(() => console.log('DB connected!'))
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
const mongoose = require('mongoose');

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
const mongoose = require('mongoose');

const PinSchema = new mongoose.Schema({
  title: String,
  content: String,
  image: String,
  latitude: Number,
  longitude: Number,
  author: { type: mongoose.Schema.ObjectId, ref: 'User'},
  comments: [
    {
      text: String,
      createdAt: { type: Date, default: Date.now },
      author: { type: mongoose.Schema.ObjectId, ref: 'User'}
    }
  ]
}, { timestamps: true });

module.exports = mongoose.model('Pin', PinSchema);
```

### Section 3: Social Login with Google OAuth 2.0

**7. Exploring our React App**

When you're inside client folder, run

$ npm run dev

**8. Setting up Google OAuth**

Login to concole.developer.google.com. Create a new project, enable Google+ API.
Create credentials.

Add client ID to the .env file

.env
```javascript
//...

OAUTH_CLIENT_ID=29144684875-ofvjb22rtvd1t90bn0db34d91r4l2e84.apps.googleusercontent.com
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
      clientId='29144684875-ofvjb22rtvd1t90bn0db34d91r4l2e84.apps.googleusercontent.com'
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

import { GraphQLClient } from 'graphql-request';

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

    const client = new GraphQLClient('http://localhost:4000/graphql', {
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
const { findOrCreateUser } = require('./controllers/userController');

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
const User = require('../models/User');
const { OAuth2Client } = require('google-auth-library');

const client = new OAuth2Client(process.env.OAUTH_CLIENT_ID);

exports.findOrCreateUser = async token => {
  // verify auth token
  const googleUser = await verifyAuthToken(token);
  // check if the user exists
  const user = await checkIfUserExists(googleUser.email);
  // if user exists,return them; otherwise, create a new user
  return user ? user : createNewUser(googleUser);
}

const verifyAuthToken = async token => {
  try {
    const ticket = await client.verifyIdToken({
      idToken: token,
      audience: process.env.OAUTH_CLIENT_ID
    });
    return ticket.getPayload();
  } catch (err) {
    console.error('Error verifying auth token', err);
  }
};

const checkIfUserExists = async email => await User.findOne({ email }).exec();

const createNewUser = googleUser => {
  const { name, email, picture } = googleUser;
  const user = { name, email, picture };
  return new User(user).save(); // Add a new User to database
}
```

resolvers.js
```javascript
const { AuthenticationError } = require('apollo-server');

// ...

const authenticated = next => (root, args, ctx, info) => {
  if (!ctx.currentUser) {
    throw new AuthenticationError('You must be logged in.')
  }
  return next(root, args, ctx, info);
}

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
import { createContext } from 'react';

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
    case 'LOGIN_USER':
      return {
        ...state,
        currentUser: action.payload
      }
    default:
      return state;
  }
}

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
      type: 'LOGIN_USER',
      payload: data.me
    })
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

## Part 2: Feature

### Section 6: Building the Header

**14. Building Header Component**
**15. Build Signout Button**

### Section 7: Building the Map / User Geolocation

**16. Creating and Styling our Map**
**17. Placing a Pin at User's Current Position**

### Section 8: Creating Blog Area / Adding Draft Pins

**18. Adding Draft Pin**
**19. Adding Blog Area for Pin Content**
**20. Building / Styling Blog Components**
**21. Managing Pin Content State and Deleting Draft Pins**

### Section 9: Image Uploads with Cloudinary Web API

**22. Uploading Images with Cloudinary**

## Part 3: More Feature

### Section 10: Creating New User Pins

**23. Creating New Pins with CREATE_PIN Mutation**

### Section 11: Making Costom useClient Hook

**24. Create Costom GraphQL Request Hook**

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