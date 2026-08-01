---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket routing, and predictive insights
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics with user management"
  - "configure JWT authentication for user management"
  - "implement AI-based ticket classification"
  - "set up MongoDB for user and task management"
  - "create a kanban board with task tracking"
  - "deploy the enterprise user management system"
  - "integrate ML service for risk prediction"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System with AI Analytics, a full-stack JavaScript/Python application for managing users, tasks, and support tickets with intelligent AI-driven insights including risk detection, anomaly detection, burnout analysis, and predictive analytics.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control (RBAC) with JWT authentication
- **Task Management**: Kanban-style boards with time tracking and progress monitoring
- **Support Tickets**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Admin Dashboard**: Organization-wide analytics, audit logs, and user monitoring
- **ML Service**: FastAPI-based microservice using scikit-learn and River for online learning

## Installation

### Prerequisites
- Node.js (v14+)
- Python 3.8+
- MongoDB

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install
```

Create `.env` file in backend directory:
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:
```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:
```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:
```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Key API Endpoints

### Authentication

```javascript
// Login
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "password123"
}

// Register
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "password123",
  "role": "user"
}

// Get Current User
GET /api/auth/me
Headers: { Authorization: "Bearer <token>" }
```

### User Management (Admin)

```javascript
// Get All Users
GET /api/users
Headers: { Authorization: "Bearer <admin_token>" }

// Create User
POST /api/users
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update User
PUT /api/users/:id
{
  "name": "Jane Doe",
  "role": "manager"
}

// Delete User
DELETE /api/users/:id
```

### Task Management

```javascript
// Get User Tasks
GET /api/tasks
Headers: { Authorization: "Bearer <token>" }

// Create Task
POST /api/tasks
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update Task Status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
}

// Track Time
POST /api/tasks/:id/time
{
  "duration": 3600,
  "action": "start"
}
```

### Support Tickets

```javascript
// Create Ticket
POST /api/tickets
{
  "title": "Cannot login",
  "description": "Getting authentication error",
  "priority": "high",
  "category": "technical"
}

// Get Tickets
GET /api/tickets
Query params: ?status=open&priority=high

// Update Ticket
PATCH /api/tickets/:id
{
  "status": "in-progress",
  "assignedTo": "admin_id"
}
```

### AI Analytics Endpoints

```javascript
// Risk Prediction
POST /api/ai/predict-risk
{
  "userId": "user_id",
  "metrics": {
    "failedLogins": 3,
    "taskCompletionRate": 0.45,
    "avgResponseTime": 150
  }
}

// Burnout Detection
POST /api/ai/detect-burnout
{
  "userId": "user_id",
  "workload": {
    "tasksAssigned": 25,
    "hoursWorked": 60,
    "weekNumber": 15
  }
}

// Ticket Classification
POST /api/ai/classify-ticket
{
  "title": "Server down",
  "description": "Production server is not responding"
}
```

## Real Code Examples

### Frontend: User Login Component

```javascript
// frontend/src/components/Login.jsx
import React, { useState } from 'react';
import axios from 'axios';

const Login = () => {
  const [credentials, setCredentials] = useState({ email: '', password: '' });
  const [error, setError] = useState('');

  const handleLogin = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_API_URL}/api/auth/login`,
        credentials
      );
      
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
      
      window.location.href = response.data.user.role === 'admin' 
        ? '/admin/dashboard' 
        : '/user/dashboard';
    } catch (err) {
      setError(err.response?.data?.message || 'Login failed');
    }
  };

  return (
    <form onSubmit={handleLogin}>
      <input
        type="email"
        placeholder="Email"
        value={credentials.email}
        onChange={(e) => setCredentials({ ...credentials, email: e.target.value })}
        required
      />
      <input
        type="password"
        placeholder="Password"
        value={credentials.password}
        onChange={(e) => setCredentials({ ...credentials, password: e.target.value })}
        required
      />
      {error && <p className="error">{error}</p>}
      <button type="submit">Login</button>
    </form>
  );
};

export default Login;
```

### Frontend: Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="kanban-board">
      <div className="column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <TaskCard key={task._id} task={task} onStatusChange={updateTaskStatus} />
        ))}
      </div>
      <div className="column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <TaskCard key={task._id} task={task} onStatusChange={updateTaskStatus} />
        ))}
      </div>
      <div className="column">
        <h3>Done</h3>
        {tasks.done.map(task => (
          <TaskCard key={task._id} task={task} onStatusChange={updateTaskStatus} />
        ))}
      </div>
    </div>
  );
};
```

### Backend: JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        message: `User role ${req.user.role} is not authorized to access this route`
      });
    }
    next();
  };
};
```

### Backend: User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Generate JWT Token
const generateToken = (id) => {
  return jwt.sign({ id }, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_EXPIRE
  });
};

// Register User
exports.register = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;

    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const user = await User.create({ name, email, password, role });

    res.status(201).json({
      success: true,
      token: generateToken(user._id),
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Login User
exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;

    if (!email || !password) {
      return res.status(400).json({ message: 'Please provide email and password' });
    }

    const user = await User.findOne({ email }).select('+password');

    if (!user || !(await user.matchPassword(password))) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    // Log successful login
    user.lastLogin = Date.now();
    await user.save();

    res.json({
      success: true,
      token: generateToken(user._id),
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Get All Users (Admin)
exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ success: true, count: users.length, data: users });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### Backend: Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

// Create Task
exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });

    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Get User Tasks
exports.getTasks = async (req, res) => {
  try {
    let query = {};
    
    if (req.user.role !== 'admin') {
      query = { $or: [{ assignedTo: req.user.id }, { createdBy: req.user.id }] };
    }

    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name email')
      .sort('-createdAt');

    res.json({ success: true, count: tasks.length, data: tasks });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

// Update Task Status
exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    task.status = req.body.status;
    
    if (req.body.status === 'done') {
      task.completedAt = Date.now();
    }

    await task.save();

    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### ML Service: AI Analytics (FastAPI)

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
from river import linear_model, preprocessing
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Models
risk_model = None
anomaly_detector = None
burnout_predictor = None

class RiskPredictionRequest(BaseModel):
    userId: str
    failedLogins: int
    taskCompletionRate: float
    avgResponseTime: float
    accessPatternScore: Optional[float] = 0.5

class BurnoutRequest(BaseModel):
    userId: str
    tasksAssigned: int
    hoursWorked: float
    weekNumber: int
    completionRate: float

class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    priority: Optional[str] = "medium"

@app.on_event("startup")
async def load_models():
    global risk_model, anomaly_detector, burnout_predictor
    
    model_path = os.getenv("MODEL_PATH", "./models")
    
    # Initialize or load models
    try:
        risk_model = joblib.load(f"{model_path}/risk_model.pkl")
    except:
        risk_model = RandomForestClassifier(n_estimators=100, random_state=42)
    
    try:
        anomaly_detector = joblib.load(f"{model_path}/anomaly_detector.pkl")
    except:
        anomaly_detector = IsolationForest(contamination=0.1, random_state=42)
    
    burnout_predictor = linear_model.LogisticRegression()

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user security risk based on behavior patterns"""
    try:
        features = np.array([[
            request.failedLogins,
            request.taskCompletionRate,
            request.avgResponseTime,
            request.accessPatternScore
        ]])
        
        if hasattr(risk_model, 'predict_proba'):
            risk_score = float(risk_model.predict_proba(features)[0][1])
        else:
            # Train with dummy data if model not trained
            X_dummy = np.random.rand(100, 4)
            y_dummy = np.random.randint(0, 2, 100)
            risk_model.fit(X_dummy, y_dummy)
            risk_score = float(risk_model.predict_proba(features)[0][1])
        
        risk_level = "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        
        return {
            "userId": request.userId,
            "riskScore": round(risk_score, 3),
            "riskLevel": risk_level,
            "recommendations": get_risk_recommendations(risk_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: RiskPredictionRequest):
    """Detect anomalous user behavior"""
    try:
        features = np.array([[
            request.failedLogins,
            request.taskCompletionRate,
            request.avgResponseTime,
            request.accessPatternScore
        ]])
        
        if not hasattr(anomaly_detector, 'offset_'):
            # Fit with dummy data
            X_dummy = np.random.rand(100, 4)
            anomaly_detector.fit(X_dummy)
        
        prediction = anomaly_detector.predict(features)[0]
        anomaly_score = anomaly_detector.score_samples(features)[0]
        
        is_anomaly = prediction == -1
        
        return {
            "userId": request.userId,
            "isAnomaly": bool(is_anomaly),
            "anomalyScore": float(anomaly_score),
            "alert": "Suspicious activity detected" if is_anomaly else "Normal behavior"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutRequest):
    """Predict employee burnout risk"""
    try:
        # Online learning approach
        features = {
            'tasks': request.tasksAssigned,
            'hours': request.hoursWorked,
            'week': request.weekNumber,
            'completion': request.completionRate
        }
        
        # Calculate burnout score
        workload_factor = (request.tasksAssigned / 20) * 0.4
        hours_factor = (request.hoursWorked / 40) * 0.3
        completion_factor = (1 - request.completionRate) * 0.3
        
        burnout_score = min(workload_factor + hours_factor + completion_factor, 1.0)
        
        burnout_level = "high" if burnout_score > 0.7 else "medium" if burnout_score > 0.4 else "low"
        
        return {
            "userId": request.userId,
            "burnoutScore": round(burnout_score, 3),
            "burnoutLevel": burnout_level,
            "recommendations": get_burnout_recommendations(burnout_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket and suggest routing"""
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple keyword-based classification
        categories = {
            "technical": ["error", "bug", "crash", "server", "database", "api"],
            "access": ["login", "password", "permission", "access", "authentication"],
            "feature": ["request", "feature", "enhancement", "suggestion"],
            "billing": ["payment", "invoice", "subscription", "billing"]
        }
        
        scores = {}
        for category, keywords in categories.items():
            score = sum(1 for keyword in keywords if keyword in text)
            scores[category] = score
        
        category = max(scores, key=scores.get) if max(scores.values()) > 0 else "general"
        
        # Suggest department routing
        routing = {
            "technical": "Engineering",
            "access": "IT Support",
            "feature": "Product",
            "billing": "Finance",
            "general": "Customer Support"
        }
        
        return {
            "category": category,
            "suggestedDepartment": routing[category],
            "priority": request.priority,
            "confidence": round(max(scores.values()) / len(categories), 2)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_risk_recommendations(level):
    recommendations = {
        "high": [
            "Reset password immediately",
            "Enable two-factor authentication",
            "Review recent access logs",
            "Limit account privileges temporarily"
        ],
        "medium": [
            "Monitor account activity",
            "Update security settings",
            "Review connected devices"
        ],
        "low": [
            "Continue normal monitoring",
            "Keep security practices up to date"
        ]
    }
    return recommendations.get(level, [])

def get_burnout_recommendations(level):
    recommendations = {
        "high": [
            "Reduce workload immediately",
            "Schedule time off",
            "Redistribute tasks to team",
            "Schedule wellness check-in"
        ],
        "medium": [
            "Monitor workload closely",
            "Encourage regular breaks",
            "Review task priorities"
        ],
        "low": [
            "Maintain current workload",
            "Continue healthy work habits"
        ]
    }
    return recommendations.get(level, [])

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "Enterprise User Management ML Service"}
```

### MongoDB Models

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please add a name']
  },
  email: {
    type: String,
    required: [true, 'Please add an email'],
    unique: true,
    match: [/^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/, 'Please add a valid email']
  },
  password: {
    type: String,
    required: [true, 'Please add a password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: {
    type: String
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  lastLogin: {
    type: Date
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Encrypt password before saving
UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) {
    next();
  }
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

// Match password
UserSchema.methods.matchPassword = async function(enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: [true, 'Please add a task title']
  },
  description: {
    type: String
  },
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  assignedTo: {
    type: mongoose.Schema.ObjectId,
    ref: 'User'
  },
  createdBy: {
    type: mongoose.Schema.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: {
    type: Date
  },
  timeTracked: {
    type: Number,
    default: 0
  },
  completedAt: {
    type: Date
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', TaskSchema);
```

## Common Patterns

### Protected Routes Pattern

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');
const {
  getTasks,
  createTask,
  updateTask,
  deleteTask,
  updateTaskStatus
} = require('../controllers/taskController');

router.route('/')
  .get(protect, getTasks)
  .post(protect, authorize('admin', 'manager'), createTask);

router.route('/:id')
  .put(protect, updateTask)
  .delete(protect, authorize('admin'), deleteTask);

router.patch('/:id/status', protect, updateTaskStatus);

module.exports = router;
```

### Axios Instance with Interceptors

```javascript
// frontend/src/utils/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

// Request interceptor to add token
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor for error handling
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### AI Integration Pattern

```javascript
// frontend/src/utils/aiService.js
import axios from 'axios';

const mlApi = axios.create({
  baseURL: process.env.REACT_APP_ML_SERVICE_URL
});

export const predictUserRisk = async (userId, metrics) => {
  try {
    const response = await mlApi.post('/predict-risk', {
      userId,
      ...metrics
    });
    return response.data;
  } catch (error) {
    console.error('Risk prediction failed:', error);
    throw error;
  }
};

export const detectBurnout = async (userId, workload) => {
  try {
    const response = await mlApi.post('/detect-burnout', {
      userId,
      ...workload
    });
    return response.data;
  } catch (error) {
    console.error('Burnout detection failed:', error);
    throw error;
  }
};

export const classifyTicket = async (ticket) => {
  try {
    const response = await mlApi.post('/classify-ticket', ticket);
    return response.data;
  } catch (error) {
    console.error('Ticket classification failed:', error);
    throw error;
  }
};
```

## Configuration

### Backend Environment Variables

```bash
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt

# JWT
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
```

### Frontend Environment Variables

```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
REACT_APP_VERSION=1.0.0
```

### ML Service Environment Variables

```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
MAX_WORKERS=4
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Configuration

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');

const app = express();

app.use(cors({
  origin: process.env.NODE_ENV === 'production' 
    ? 'https://your-frontend-domain.com' 
    : 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Not Loading Models

```python
# ml-service/main.py
import
