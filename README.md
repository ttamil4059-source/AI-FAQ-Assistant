1. Project Structure
AI-FAQ-Assistant/
│
├── src/
│   ├── config/
│   │   └── db.js
│   │
│   ├── controllers/
│   │   ├── authController.js
│   │   └── faqController.js
│   │
│   ├── middleware/
│   │   ├── authMiddleware.js
│   │   └── errorMiddleware.js
│   │
│   ├── models/
│   │   ├── User.js
│   │   └── FAQ.js
│   │
│   ├── routes/
│   │   ├── authRoutes.js
│   │   └── faqRoutes.js
│   │
│   └── app.js
│
├── .env
├── .env.example
├── .gitignore
├── package.json
├── server.js
└── README.md

2. package.json
{
  "name": "ai-faq-assistant",
  "version": "1.0.0",
  "description": "AI FAQ Assistant backend using Node.js, Express.js and MongoDB",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.21.0",
    "jsonwebtoken": "^9.0.2",
    "mongoose": "^8.7.0"
  },
  "devDependencies": {
    "nodemon": "^3.1.7"
  }
}

3. server.js
require("dotenv").config();

const app = require("./src/app");
const connectDB = require("./src/config/db");

const PORT = process.env.PORT || 5000;

const startServer = async () => {
  try {
    await connectDB();

    app.listen(PORT, () => {
      console.log(`Server running on port ${PORT}`);
      console.log(`http://localhost:${PORT}`);
    });
  } catch (error) {
    console.error("Failed to start server:", error.message);
    process.exit(1);
  }
};

startServer();

4. src/app.js
const express = require("express");
const cors = require("cors");

const authRoutes = require("./routes/authRoutes");
const faqRoutes = require("./routes/faqRoutes");
const errorHandler = require("./middleware/errorMiddleware");

const app = express();

app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

app.get("/", (req, res) => {
  res.json({
    success: true,
    message: "AI FAQ Assistant API is running"
  });
});

app.use("/api/auth", authRoutes);
app.use("/api/faqs", faqRoutes);

app.use(errorHandler);

module.exports = app;

5. src/config/db.js
const mongoose = require("mongoose");

const connectDB = async () => {
  try {
    const connection = await mongoose.connect(process.env.MONGO_URI);

    console.log(`MongoDB connected: ${connection.connection.host}`);
  } catch (error) {
    console.error("MongoDB connection error:", error.message);
    throw error;
  }
};

module.exports = connectDB;

6. src/models/User.js
const mongoose = require("mongoose");
const bcrypt = require("bcryptjs");

const userSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: true,
      trim: true
    },

    email: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
      trim: true
    },

    password: {
      type: String,
      required: true,
      minlength: 6
    }
  },
  {
    timestamps: true
  }
);

userSchema.pre("save", async function (next) {
  if (!this.isModified("password")) {
    return next();
  }

  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);

  next();
});

userSchema.methods.comparePassword = async function (password) {
  return bcrypt.compare(password, this.password);
};

module.exports = mongoose.model("User", userSchema);

7. src/models/FAQ.js
const mongoose = require("mongoose");

const faqSchema = new mongoose.Schema(
  {
    question: {
      type: String,
      required: true,
      trim: true
    },

    answer: {
      type: String,
      required: true,
      trim: true
    },

    category: {
      type: String,
      default: "General",
      trim: true
    },

    keywords: {
      type: [String],
      default: []
    },

    createdBy: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
      required: true
    }
  },
  {
    timestamps: true
  }
);

module.exports = mongoose.model("FAQ", faqSchema);

8. src/controllers/authController.js
const jwt = require("jsonwebtoken");
const User = require("../models/User");

const generateToken = (id) => {
  return jwt.sign(
    { id },
    process.env.JWT_SECRET,
    {
      expiresIn: "7d"
    }
  );
};

const register = async (req, res, next) => {
  try {
    const { name, email, password } = req.body;

    if (!name || !email || !password) {
      return res.status(400).json({
        success: false,
        message: "Name, email and password are required"
      });
    }

    const existingUser = await User.findOne({ email });

    if (existingUser) {
      return res.status(409).json({
        success: false,
        message: "User already exists"
      });
    }

    const user = await User.create({
      name,
      email,
      password
    });

    res.status(201).json({
      success: true,
      message: "Registration successful",
      token: generateToken(user._id),
      user: {
        id: user._id,
        name: user.name,
        email: user.email
      }
    });
  } catch (error) {
    next(error);
  }
};

const login = async (req, res, next) => {
  try {
    const { email, password } = req.body;

    const user = await User.findOne({ email });

    if (!user || !(await user.comparePassword(password))) {
      return res.status(401).json({
        success: false,
        message: "Invalid email or password"
      });
    }

    res.json({
      success: true,
      message: "Login successful",
      token: generateToken(user._id),
      user: {
        id: user._id,
        name: user.name,
        email: user.email
      }
    });
  } catch (error) {
    next(error);
  }
};

module.exports = {
  register,
  login
};

9. src/controllers/faqController.js
const FAQ = require("../models/FAQ");

const createFAQ = async (req, res, next) => {
  try {
    const {
      question,
      answer,
      category,
      keywords
    } = req.body;

    if (!question || !answer) {
      return res.status(400).json({
        success: false,
        message: "Question and answer are required"
      });
    }

    const faq = await FAQ.create({
      question,
      answer,
      category,
      keywords,
      createdBy: req.user.id
    });

    res.status(201).json({
      success: true,
      message: "FAQ created successfully",
      faq
    });
  } catch (error) {
    next(error);
  }
};

const getFAQs = async (req, res, next) => {
  try {
    const { search, category } = req.query;

    const filter = {};

    if (category) {
      filter.category = category;
    }

    if (search) {
      filter.$or = [
        {
          question: {
            $regex: search,
            $options: "i"
          }
        },
        {
          answer: {
            $regex: search,
            $options: "i"
          }
        },
        {
          keywords: {
            $regex: search,
            $options: "i"
          }
        }
      ];
    }

    const faqs = await FAQ.find(filter)
      .populate("createdBy", "name email")
      .sort({ createdAt: -1 });

    res.json({
      success: true,
      count: faqs.length,
      faqs
    });
  } catch (error) {
    next(error);
  }
};

const getFAQById = async (req, res, next) => {
  try {
    const faq = await FAQ.findById(req.params.id)
      .populate("createdBy", "name email");

    if (!faq) {
      return res.status(404).json({
        success: false,
        message: "FAQ not found"
      });
    }

    res.json({
      success: true,
      faq
    });
  } catch (error) {
    next(error);
  }
};

const updateFAQ = async (req, res, next) => {
  try {
    const faq = await FAQ.findById(req.params.id);

    if (!faq) {
      return res.status(404).json({
        success: false,
        message: "FAQ not found"
      });
    }

    faq.question = req.body.question ?? faq.question;
    faq.answer = req.body.answer ?? faq.answer;
    faq.category = req.body.category ?? faq.category;
    faq.keywords = req.body.keywords ?? faq.keywords;

    await faq.save();

    res.json({
      success: true,
      message: "FAQ updated successfully",
      faq
    });
  } catch (error) {
    next(error);
  }
};

const deleteFAQ = async (req, res, next) => {
  try {
    const faq = await FAQ.findByIdAndDelete(req.params.id);

    if (!faq) {
      return res.status(404).json({
        success: false,
        message: "FAQ not found"
      });
    }

    res.json({
      success: true,
      message: "FAQ deleted successfully"
    });
  } catch (error) {
    next(error);
  }
};

module.exports = {
  createFAQ,
  getFAQs,
  getFAQById,
  updateFAQ,
  deleteFAQ
};

10. src/middleware/authMiddleware.js
const jwt = require("jsonwebtoken");
const User = require("../models/User");

const protect = async (req, res, next) => {
  try {
    const authHeader = req.headers.authorization;

    if (!authHeader || !authHeader.startsWith("Bearer ")) {
      return res.status(401).json({
        success: false,
        message: "Authentication required"
      });
    }

    const token = authHeader.split(" ")[1];

    const decoded = jwt.verify(
      token,
      process.env.JWT_SECRET
    );

    const user = await User.findById(decoded.id)
      .select("-password");

    if (!user) {
      return res.status(401).json({
        success: false,
        message: "User not found"
      });
    }

    req.user = user;

    next();
  } catch (error) {
    return res.status(401).json({
      success: false,
      message: "Invalid or expired token"
    });
  }
};

module.exports = protect;

11. src/middleware/errorMiddleware.js
const errorHandler = (err, req, res, next) => {
  console.error(err.stack);

  res.status(err.statusCode || 500).json({
    success: false,
    message: err.message || "Internal server error"
  });
};

module.exports = errorHandler;

12. src/routes/authRoutes.js
const express = require("express");

const {
  register,
  login
} = require("../controllers/authController");

const router = express.Router();

router.post("/register", register);
router.post("/login", login);

module.exports = router;

13. src/routes/faqRoutes.js
const express = require("express");

const protect = require("../middleware/authMiddleware");

const {
  createFAQ,
  getFAQs,
  getFAQById,
  updateFAQ,
  deleteFAQ
} = require("../controllers/faqController");

const router = express.Router();

router.get("/", getFAQs);
router.get("/:id", getFAQById);

router.post("/", protect, createFAQ);
router.put("/:id", protect, updateFAQ);
router.delete("/:id", protect, deleteFAQ);

module.exports = router;

14. .env.example
PORT=5000

MONGO_URI=mongodb://127.0.0.1:27017/ai_faq_assistant

JWT_SECRET=your_super_secret_jwt_key

Create your actual .env:

PORT=5000
MONGO_URI=mongodb://127.0.0.1:27017/ai_faq_assistant
JWT_SECRET=my_secret_key_123456

Do not upload .env to GitHub.

15. .gitignore
node_modules/
.env
.DS_Store
npm-debug.log*

16. README.md
# AI FAQ Assistant

An AI-powered FAQ Assistant backend built with Node.js, Express.js,
MongoDB and Mongoose.

The application provides user authentication, FAQ management,
search functionality and a foundation for integrating AI-based
FAQ generation and semantic search.

---

## Features

- User registration
- User login
- JWT authentication
- Password hashing using bcrypt
- FAQ creation
- FAQ retrieval
- FAQ update
- FAQ deletion
- FAQ search
- FAQ category filtering
- MongoDB database
- Mongoose ODM
- Centralized error handling
- REST API architecture
- CORS support
- Environment-based configuration

---

## Technologies

| Technology | Purpose |
|------------|---------|
| Node.js | Backend runtime |
| Express.js | REST API framework |
| MongoDB | Database |
| Mongoose | MongoDB ODM |
| JWT | Authentication |
| bcryptjs | Password hashing |
| dotenv | Environment variables |
| CORS | Cross-origin requests |

---

## Project Structure

```text
AI-FAQ-Assistant/
│
├── src/
│   ├── config/
│   │   └── db.js
│   ├── controllers/
│   │   ├── authController.js
│   │   └── faqController.js
│   ├── middleware/
│   │   ├── authMiddleware.js
│   │   └── errorMiddleware.js
│   ├── models/
│   │   ├── User.js
│   │   └── FAQ.js
│   ├── routes/
│   │   ├── authRoutes.js
│   │   └── faqRoutes.js
│   └── app.js
│
├── .env
├── .env.example
├── .gitignore
├── package.json
├── server.js
└── README.md

Requirements
Install the following:

Node.js

npm

MongoDB

Check Node.js:

node --version

Check npm:

npm --version

Installation
Clone the repository:

git clone <your-repository-url>

Move into the project:

cd AI-FAQ-Assistant

Install dependencies:

npm install

Environment Configuration
Create a .env file in the project root.

PORT=5000
MONGO_URI=mongodb://127.0.0.1:27017/ai_faq_assistant
JWT_SECRET=your_super_secret_key

For MongoDB Atlas, replace MONGO_URI with your Atlas connection string.

Run the Application
Development mode:

npm run dev

Production/start mode:

npm start

The server will run at:

http://localhost:5000

API Documentation
Authentication
Register
POST /api/auth/register

Request:

{
  "name": "John",
  "email": "john@example.com",
  "password": "password123"
}

Response:

{
  "success": true,
  "message": "Registration successful",
  "token": "JWT_TOKEN",
  "user": {
    "id": "USER_ID",
    "name": "John",
    "email": "john@example.com"
  }
}

Login
POST /api/auth/login

Request:

{
  "email": "john@example.com",
  "password": "password123"
}

FAQ APIs
Get All FAQs
GET /api/faqs

Search FAQs
GET /api/faqs?search=payment

Filter by Category
GET /api/faqs?category=General

Get FAQ by ID
GET /api/faqs/:id

Create FAQ
Authentication required.

POST /api/faqs

Header:

Authorization: Bearer YOUR_JWT_TOKEN

Request:

{
  "question": "How can I reset my password?",
  "answer": "Open the login page and select Forgot Password.",
  "category": "Account",
  "keywords": [
    "password",
    "reset",
    "account"
  ]
}

Update FAQ
Authentication required.

PUT /api/faqs/:id

Request:

{
  "answer": "Use the Forgot Password option on the login page."
}

Delete FAQ
Authentication required.

DELETE /api/faqs/:id

Authentication
Protected endpoints require a JWT token.

Example:

Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

The token is generated after successful registration or login.

Database
The project uses MongoDB.

Database:

ai_faq_assistant

Collections:

users
faqs

Future AI Features
The backend can be extended with:

AI-generated FAQ questions

AI-generated answers

Semantic FAQ search

Context-aware responses

Document/PDF ingestion

Embeddings

Vector database integration

Chat-based FAQ assistant

Conversation history

FAQ confidence scoring

Automatic FAQ categorization

Security
The application includes:

JWT authentication

Password hashing

Protected routes

Environment variables

Centralized error handling

No password returned in authentication responses

Production deployments should additionally use:

HTTPS

Rate limiting

Request validation

Security headers

Input sanitization

Secret management

Database access controls

Testing
The APIs can be tested using:

Postman

Thunder Client

Insomnia

curl

Example:

curl http://localhost:5000/

Expected response:

{
  "success": true,
  "message": "AI FAQ Assistant API is running"
}

Team Development
Recommended Git workflow:

git checkout -b feature/authentication

Make changes and commit:

git add .
git commit -m "Add JWT authentication"

Push:

git push origin feature/authentication

Create a Pull Request for review.

License
This project is developed for educational and project purposes.


### Run it

After creating the files:

```bash
npm install
npm run dev

Then test:

http://localhost:5000/

This gives you the coding + README foundation matching the Node.js/Express/MongoDB project described in the screenshot. The next logical implementation is the actual AI layer—FAQ generation, semantic search, and context-b
