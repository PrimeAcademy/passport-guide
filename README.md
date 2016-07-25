*Note: this guide assumes you already have a basic express web server with body-parser set up*

Passport is authentication middleware for Node.js.

It’s extremely flexible and modular and can be dropped in to any Express-based web application, and can authenticate users via many different authentication mechanisms called “strategies”.

Strategies are packaged as individual modules so you can choose which strategies to employ, without creating unnecessary dependencies.


Let’s add Passport to our application.

```
npm install passport --save
```

To use it, we need to require it in server.js (our main application file).

```
var passport = require('passport');
```

Passport uses sessions. Session provides a way to identify a user across more than one page request or visit to a Web site and to store information about that user.


We have to install and use express-session:

```
npm install express-session --save
```

Make sure these lines go before you use your routes in server.js.

```
var session = require('express-session');

app.use(session({
   secret: 'secret',
   key: 'user',
   resave: true,
   saveUninitialized: false,
   cookie: { maxAge: 60000, secure: false }
}));
```

The most widely used way for websites to authenticate users is via a username and password. Support for this mechanism is provided by the “passport-local” module.


Let’s add Passport-Local to our application:

```
npm install passport-local --save
```

To use it, we need to require it in server.js, and get a reference to the module’s strategy object. Make sure theses lines go after those you just typed, but before you use your routes!

```
var localStrategy = require('passport-local').Strategy;
```

We need to initialize passport:

```
app.use(passport.initialize());
app.use(passport.session());
```

Now, we have to tell passport which strategy to use inside our server.js file.

```
passport.use('local', new localStrategy({ passReqToCallback : true, usernameField: 'username' },
  function(req, username, password, done) {

    // our implementation will go here

  }
));
```

The verify callback for local authentication accepts username and password arguments, which are submitted to the application via a login form. Inside this form we’ll authenticate users. However, we don’t have users set up correctly yet.

So, we need to create a user model in Mongo. Make sure you have Mongoose installed.

```
npm install mongoose --save
```

Require mongoose and add a mongo connection to our server.js file (give it a unique document store name). Make sure these lines go before you use your models in server.js.

```
var mongoose = require('mongoose');

// Mongo setup
var mongoURI = "mongodb://localhost:27017/prime_example_passport";
var MongoDB = mongoose.connect(mongoURI).connection;

MongoDB.on('error', function (err) {
   console.log('mongodb connection error', err);
});

MongoDB.once('open', function () {
 console.log('mongodb connection open');
});
```

Next, we’re going to add a user.js file to the models folder.

```
var mongoose = require('mongoose'),
    Schema = mongoose.Schema;

var UserSchema = new Schema({
   username: { type: String, required: true, index: { unique: true } },
   password: { type: String, required: true }
});

module.exports = mongoose.model('User', UserSchema);
```

Also create a convenience method for comparing passwords later on.

```
UserSchema.methods.comparePassword = function(candidatePassword, cb) {
    cb(null, this.password == candidatePassword);
  });
};
```

The user’s password plus some extra random “salt” is sent through a one-way function to compute a hash. This way each user’s password is uniquely encrypted.


Back in server.js we can add the rest of our authentication strategy.  Require our newly created user in server.js

```
var User = require('./models/user');
```

Then create the rest of the function for authenticating users. Serialize and deserialize <sup id="a1">[1](#f1)</sup> allow user information to be stored and retrieved from session.

```
passport.serializeUser(function(user, done) {
   done(null, user.id);
});

passport.deserializeUser(function(id, done) {
  User.findById(id, function(err,user){
    if(err) {
      return done(err);
    }
    done(null,user);
  });
});
```

NOTE: You need to replace the other passport.use('local'... with this completed version:

```
passport.use('local', new localStrategy({
      passReqToCallback : true,
      usernameField: 'username'
  },
  function(req, username, password, done){
    User.findOne({ username: username }, function(err, user) {
      if (err) {
         throw err
      };

      if (!user) {
        return done(null, false, {message: 'Incorrect username and password.'});
      }

      // test a matching password
      user.comparePassword(password, function(err, isMatch) {
        if (err) {
          throw err;
        }

        if (isMatch) {
          return done(null, user);
        } else {
          done(null, false, { message: 'Incorrect username and password.' });
        }
      });
    });
  })
);
```

Next create an login.html page and add a login form.

```
<form action="/login" method="post">
   <div>
       <label for="username">Username:</label>
       <input type="text" name="username" id="username"/>
   </div>
   <div>
       <label for="password">Password:</label>
       <input type="password" name="password" id="password"/>
   </div>
   <div>
       <input type="submit" value="Log In"/>
       <a href="/register">Register</a>
   </div>
</form>
```

Create a route to handle logging in. Passport.authenticate is specifying our ‘local’ strategy that we created, and specifies a failure and success redirect.

**NOTE: If you have not put BOTH body parser functions (json and urlencoded) do so now!**

```
var express = require('express');
var router = express.Router();
var passport = require('passport');
var path = require('path');


router.get("/", function(req,res,next){
  res.sendFile(path.resolve(__dirname, '../views/login.html'));
});


router.post('/',
  passport.authenticate('local', {
    successRedirect: '/views/success.html',
    failureRedirect: '/views/failure.html'
  })
);


module.exports = router;
```

We also need a way for users to register. Create a register.html file with the following form in it:

```
<form action="/register" method="post">
   <div>
       <label for="username">Username:</label>
       <input type="text" name="username" id="username"/>
   </div>
   <div>
       <label for="password">Password:</label>
       <input type="password" name="password" id="password"/>
   </div>
   <div>
       <input type="submit" value="Register"/>
   </div>
</form>
```

Also create a register.js route file. Remember, the pre-save function will encrypt the passwords for us!

```
var express = require('express');
var router = express.Router();
var passport = require('passport');
var path = require('path');
var Users = require('../models/user');

router.get('/', function(req, res, next){
   res.sendFile(path.resolve(__dirname, '../public/views/register.html'));
});

router.post('/', function(req,res,next) {
  Users.create(req.body, function (err, post) {
    if (err) {
      next(err);
    } else {
      // we registered the user, but they haven't logged in yet.
      // redirect them to the login page
      res.redirect('/');
    }
  })
});

module.exports = router;
```

Add the login and register routes to server.js:

```
var register = require('./routes/register');
var login = require('./routes/login');

...

app.use('/register', register);
app.use('/login', login);

```

Make sure we also have a route to serve the login page at the root URL.

```
app.get("/", function(req,res,next){
  res.sendFile(path.resolve(__dirname, 'public/views/login.html'));
});
```

Finally, let’s test user.isAuthenticated() in the login.js route

```
router.get('/', function(req, res, next) {
  res.json(req.isAuthenticated());
});
```


That’s it! You have users authenticating and you’re storing unencrypted passwords. Nice job! Remember that this is **not** secure and you should never deploy an application like this. We will discuss how to encrypt passwords later.

![Bill Murray says, "Good job".](bill.jpg)

<b id="f1">1</b> Somewhat helpful description of the flow: http://stackoverflow.com/questions/27637609/understanding-passport-serialize-deserialize [↩](#a1)
