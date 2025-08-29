[![Open in Visual Studio Code](https://classroom.github.com/assets/open-in-vscode-2e0aaae1b6195c2367325f4f02e2d04e9abb55f0b24a779b69b11b9e10269abc.svg)](https://classroom.github.com/online_ide?assignment_repo_id=20212730&assignment_repo_type=AssignmentRepo)
# MERN Stack Integration Assignment

This assignment focuses on building a full-stack MERN (MongoDB, Express.js, React.js, Node.js) application that demonstrates seamless integration between front-end and back-end components.

## Assignment Overview

You will build a blog application with the following features:
1. RESTful API with Express.js and MongoDB
2. React front-end with component architecture
3. Full CRUD functionality for blog posts
4. User authentication and authorization
5. Advanced features like image uploads and comments

## Project Structure

```
mern-blog/
├── client/                 # React front-end
│   ├── public/             # Static files
│   ├── src/                # React source code
│   │   ├── components/     # Reusable components
│   │   ├── pages/          # Page components
│   │   ├── hooks/          # Custom React hooks
│   │   ├── services/       # API services
│   │   ├── context/        # React context providers
│   │   └── App.jsx         # Main application component
│   └── package.json        # Client dependencies
├── server/                 # Express.js back-end
│   ├── config/             # Configuration files
│   ├── controllers/        # Route controllers
│   ├── models/             # Mongoose models
│   ├── routes/             # API routes
│   ├── middleware/         # Custom middleware
│   ├── utils/              # Utility functions
│   ├── server.js           # Main server file
│   └── package.json        # Server dependencies
└── README.md               # Project documentation
```

## Getting Started

1. Accept the GitHub Classroom assignment invitation
2. Clone your personal repository that was created by GitHub Classroom
3. Follow the setup instructions in the `Week4-Assignment.md` file
4. Complete the tasks outlined in the assignment

## Files Included

- `Week4-Assignment.md`: Detailed assignment instructions
- Starter code for both client and server:
  - Basic project structure
  - Configuration files
  - Sample models and components

## Requirements

- Node.js (v18 or higher)
- MongoDB (local installation or Atlas account)
- npm or yarn
- Git

## Submission

Your work will be automatically submitted when you push to your GitHub Classroom repository. Make sure to:

1. Complete both the client and server portions of the application
2. Implement all required API endpoints
3. Create the necessary React components and hooks
4. Document your API and setup process in the README.md
5. Include screenshots of your working application

## Resources

- [MongoDB Documentation](https://docs.mongodb.com/)
- [Express.js Documentation](https://expressjs.com/)
- [React Documentation](https://react.dev/)
- [Node.js Documentation](https://nodejs.org/en/docs/)
- [Mongoose Documentation](https://mongoosejs.com/docs/)


// index.js (run with: node index.js)
import express from 'express';
import mongoose from 'mongoose';
import cors from 'cors';
import cookieParser from 'cookie-parser';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import multer from 'multer';
import path from 'path';
import fs from 'fs';
import { createServer } from 'http';
import { readFileSync } from 'fs';

const app = express();
const PORT = 5000;
const JWT_SECRET = 'supersecret';
const UPLOAD_DIR = './uploads';

mongoose.connect('mongodb://127.0.0.1:27017/mini_blog');

app.use(cors({ origin: 'http://localhost:5173', credentials: true }));
app.use(express.json());
app.use(cookieParser());
app.use('/uploads', express.static(UPLOAD_DIR));

// Models
const userSchema = new mongoose.Schema({
  name: String,
  email: { type: String, unique: true },
  password: String,
  role: { type: String, default: 'user' }
});
userSchema.pre('save', async function () {
  this.password = await bcrypt.hash(this.password, 10);
});
userSchema.methods.matchPassword = function (pw) {
  return bcrypt.compare(pw, this.password);
};
const User = mongoose.model('User', userSchema);

const postSchema = new mongoose.Schema({
  title: String,
  body: String,
  slug: String,
  coverImageUrl: String,
  author: { type: mongoose.Schema.Types.ObjectId, ref: 'User' }
});
const Post = mongoose.model('Post', postSchema);

const commentSchema = new mongoose.Schema({
  body: String,
  post: { type: mongoose.Schema.Types.ObjectId, ref: 'Post' },
  author: { type: mongoose.Schema.Types.ObjectId, ref: 'User' }
});
const Comment = mongoose.model('Comment', commentSchema);

// Middleware
const auth = async (req, res, next) => {
  const token = req.cookies.token;
  if (!token) return res.status(401).json({ message: 'No token' });
  const decoded = jwt.verify(token, JWT_SECRET);
  req.user = await User.findById(decoded.id);
  next();
};

// Upload
if (!fs.existsSync(UPLOAD_DIR)) fs.mkdirSync(UPLOAD_DIR);
const storage = multer.diskStorage({
  destination: (_, __, cb) => cb(null, UPLOAD_DIR),
  filename: (_, file, cb) => cb(null, Date.now() + path.extname(file.originalname))
});
const upload = multer({ storage });

// Auth routes
app.post('/api/register', async (req, res) => {
  const user = await User.create(req.body);
  res.json(user);
});
app.post('/api/login', async (req, res) => {
  const user = await User.findOne({ email: req.body.email });
  if (!user || !(await user.matchPassword(req.body.password)))
    return res.status(401).json({ message: 'Invalid credentials' });
  const token = jwt.sign({ id: user._id }, JWT_SECRET);
  res.cookie('token', token, { httpOnly: true }).json(user);
});
app.get('/api/me', auth, (req, res) => res.json(req.user));

// Post routes
app.get('/api/posts', async (req, res) => {
  const posts = await Post.find().populate('author', 'name');
  res.json(posts);
});
app.post('/api/posts', auth, async (req, res) => {
  const post = await Post.create({ ...req.body, author: req.user._id });
  res.json(post);
});

// Comment routes
app.get('/api/comments/:postId', async (req, res) => {
  const comments = await Comment.find({ post: req.params.postId }).populate('author', 'name');
  res.json(comments);
});
app.post('/api/comments/:postId', auth, async (req, res) => {
  const comment = await Comment.create({ body: req.body.body, post: req.params.postId, author: req.user._id });
  res.json(comment);
});

// Upload route
app.post('/api/upload', auth, upload.single('image'), (req, res) => {
  res.json({ url: `/uploads/${req.file.filename}` });
});

// Minimal frontend (optional)
app.get('/', (_, res) => {
  res.send(`
    <html>
      <body>
        <h1>MERN Blog</h1>
        <p>Use Postman or connect React frontend to /api endpoints.</p>
      </body>
    </html>
  `);
});

app.listen(PORT, () => console.log(`Server running on http://localhost:${PORT}`));
