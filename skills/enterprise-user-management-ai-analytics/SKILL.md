---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management using React, Node.js, and FastAPI ML service
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics into user management
  - implement risk detection and anomaly detection for users
  - create a task management system with AI insights
  - build a kanban board with time tracking
  - set up JWT authentication for user management
  - deploy user management system with ML service
  - configure AI-powered ticket classification
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System, a full-stack application that combines user/task management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control (Admin/User), authentication via JWT
- **Task Management**: Kanban board (To Do, In Progress, Done) with time tracking
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Separate admin and user interfaces with real-time insights

**Architecture**: React frontend + Node.js/Express backend + FastAPI ML service + MongoDB database

## Installation

### Prerequisites
- Node.js (v14+)
- Python 3.8+
- MongoDB (local or Atlas)

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

Create `.env` file:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_here
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

Create `.env` file:
```env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Key API Endpoints

### Authentication (Backend)

```javascript
// POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user" // or "admin"
}

// POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}

// Response
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "507f1f77bcf86cd799439011",
    "name": "John Doe",
    "email": "john@example.com",
    "role": "user"
  }
}
```

### User Management (Backend)

```javascript
// GET /api/users (Admin only)
// Headers: { "Authorization": "Bearer <token>" }

// PUT /api/users/:id
{
  "name": "John Updated",
  "email": "john.updated@example.com",
  "role": "admin"
}

// DELETE /api/users/:id (Admin only)
```

### Task Management (Backend)

```javascript
// POST /api/tasks
{
  "title": "Implement login feature",
  "description": "Add JWT authentication",
  "assignedTo": "507f1f77bcf86cd799439011",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// GET /api/tasks?userId=507f1f77bcf86cd799439011

// PUT /api/tasks/:id
{
  "status": "in-progress",
  "timeSpent": 3600 // seconds
}
```

### Support Tickets (Backend)

```javascript
// POST /api/tickets
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when accessing /dashboard",
  "priority": "high",
  "category": "technical" // or "billing", "general"
}

// GET /api/tickets
// GET /api/tickets/:id
```

### AI Analytics (ML Service)

```python
# POST /predict/risk
{
  "user_id": "507f1f77bcf86cd799439011",
  "failed_logins": 5,
  "tasks_overdue": 3,
  "avg_task_completion_time": 7200,
  "login_frequency": 2.5
}

# Response
{
  "risk_score": 0.78,
  "risk_level": "high",
  "factors": ["high_failed_logins", "overdue_tasks"]
}

# POST /predict/anomaly
{
  "user_id": "507f1f77bcf86cd799439011",
  "login_time": "2026-04-15T03:30:00Z",
  "login_location": "Unknown IP",
  "access_pattern": "unusual"
}

# POST /predict/burnout
{
  "user_id": "507f1f77bcf86cd799439011",
  "weekly_hours": 65,
  "tasks_assigned": 25,
  "tasks_completed": 10,
  "avg_response_time": 14400
}

# POST /classify/ticket
{
  "title": "Cannot login to system",
  "description": "Password reset not working"
}

# Response
{
  "category": "technical",
  "priority": "high",
  "confidence": 0.92
}
```

## Real Code Examples

### Backend: JWT Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Backend: Task Controller

```javascript
// controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      createdBy: req.user.id,
      status: 'todo'
    });

    await task.save();
    
    // Get AI prediction for task delay
    try {
      const prediction = await axios.post(
        `${process.env.ML_SERVICE_URL}/predict/delay`,
        {
          priority,
          estimated_hours: task.estimatedHours,
          assigned_user_workload: await getUserWorkload(assignedTo)
        }
      );
      task.delayProbability = prediction.data.delay_probability;
      await task.save();
    } catch (mlError) {
      console.error('ML prediction failed:', mlError);
    }

    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status, timeSpent } = req.body;

    const task = await Task.findById(id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    task.status = status;
    if (timeSpent) {
      task.timeSpent = (task.timeSpent || 0) + timeSpent;
    }
    
    if (status === 'done') {
      task.completedAt = new Date();
    }

    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### Frontend: API Service

```javascript
// src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const getAuthHeader = () => {
  const token = localStorage.getItem('token');
  return token ? { Authorization: `Bearer ${token}` } : {};
};

export const authAPI = {
  login: async (email, password) => {
    const response = await axios.post(`${API_URL}/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', response.data.token);
    return response.data;
  },
  
  register: async (userData) => {
    const response = await axios.post(`${API_URL}/auth/register`, userData);
    return response.data;
  },
  
  logout: () => {
    localStorage.removeItem('token');
  }
};

export const taskAPI = {
  getTasks: async (userId) => {
    const response = await axios.get(`${API_URL}/tasks`, {
      params: { userId },
      headers: getAuthHeader()
    });
    return response.data;
  },
  
  createTask: async (taskData) => {
    const response = await axios.post(`${API_URL}/tasks`, taskData, {
      headers: getAuthHeader()
    });
    return response.data;
  },
  
  updateTask: async (id, updates) => {
    const response = await axios.put(`${API_URL}/tasks/${id}`, updates, {
      headers: getAuthHeader()
    });
    return response.data;
  }
};

export const userAPI = {
  getUsers: async () => {
    const response = await axios.get(`${API_URL}/users`, {
      headers: getAuthHeader()
    });
    return response.data;
  },
  
  updateUser: async (id, userData) => {
    const response = await axios.put(`${API_URL}/users/${id}`, userData, {
      headers: getAuthHeader()
    });
    return response.data;
  }
};
```

### Frontend: Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const data = await taskAPI.getTasks(userId);
      const grouped = {
        todo: data.filter(t => t.status === 'todo'),
        'in-progress': data.filter(t => t.status === 'in-progress'),
        done: data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching tasks:', error);
      setLoading(false);
    }
  };

  const handleDragStart = (e, task) => {
    e.dataTransfer.setData('taskId', task._id);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    
    try {
      await taskAPI.updateTask(taskId, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{status.replace('-', ' ').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div
              key={task._id}
              className="task-card"
              draggable
              onDragStart={(e) => handleDragStart(e, task)}
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority ${task.priority}`}>
                {task.priority}
              </span>
              {task.dueDate && (
                <span className="due-date">
                  Due: {new Date(task.dueDate).toLocaleDateString()}
                </span>
              )}
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### ML Service: Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from river import linear_model, preprocessing, compose
import joblib
import os

class RiskPredictor:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        self.model = compose.Pipeline(
            preprocessing.StandardScaler(),
            linear_model.LogisticRegression()
        )
        self._load_model()
    
    def _load_model(self):
        if os.path.exists(self.model_path):
            self.model = joblib.load(self.model_path)
    
    def predict(self, features):
        """
        Predict risk score for a user
        features: dict with keys:
          - failed_logins: int
          - tasks_overdue: int
          - avg_task_completion_time: float (seconds)
          - login_frequency: float (logins per day)
        """
        risk_score = self.model.predict_proba_one(features)
        risk_level = self._classify_risk(risk_score.get(1, 0))
        
        factors = []
        if features.get('failed_logins', 0) > 3:
            factors.append('high_failed_logins')
        if features.get('tasks_overdue', 0) > 2:
            factors.append('overdue_tasks')
        if features.get('avg_task_completion_time', 0) > 10800:
            factors.append('slow_completion')
        
        return {
            'risk_score': risk_score.get(1, 0),
            'risk_level': risk_level,
            'factors': factors
        }
    
    def _classify_risk(self, score):
        if score > 0.7:
            return 'high'
        elif score > 0.4:
            return 'medium'
        return 'low'
    
    def train(self, features, label):
        """Online learning - update model with new data"""
        self.model.learn_one(features, label)
        joblib.dump(self.model, self.model_path)

risk_predictor = RiskPredictor()
```

### ML Service: FastAPI Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
from models.risk_predictor import risk_predictor
from models.anomaly_detector import anomaly_detector
from models.ticket_classifier import ticket_classifier

app = FastAPI(title="Enterprise User Management ML Service")

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_logins: int
    tasks_overdue: int
    avg_task_completion_time: float
    login_frequency: float

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    login_time: str
    login_location: str
    access_pattern: str

@app.post("/predict/risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        features = {
            'failed_logins': request.failed_logins,
            'tasks_overdue': request.tasks_overdue,
            'avg_task_completion_time': request.avg_task_completion_time,
            'login_frequency': request.login_frequency
        }
        
        prediction = risk_predictor.predict(features)
        return prediction
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/classify/ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        text = f"{request.title} {request.description}"
        result = ticket_classifier.classify(text)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict/anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        features = {
            'user_id': request.user_id,
            'login_time': request.login_time,
            'login_location': request.login_location,
            'access_pattern': request.access_pattern
        }
        
        result = anomaly_detector.detect(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

### ML Service: Ticket Classifier

```python
# ml-service/models/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
import joblib
import os

class TicketClassifier:
    def __init__(self, model_path='./models/ticket_classifier.pkl'):
        self.model_path = model_path
        self.categories = ['technical', 'billing', 'general', 'account']
        self.priority_keywords = {
            'high': ['urgent', 'critical', 'down', 'broken', 'cannot', 'error'],
            'medium': ['issue', 'problem', 'help', 'question'],
            'low': ['info', 'suggestion', 'feature', 'request']
        }
        
        self.model = Pipeline([
            ('tfidf', TfidfVectorizer(max_features=1000)),
            ('clf', MultinomialNB())
        ])
        
        self._load_or_init_model()
    
    def _load_or_init_model(self):
        if os.path.exists(self.model_path):
            self.model = joblib.load(self.model_path)
    
    def classify(self, text):
        """Classify ticket and determine priority"""
        text_lower = text.lower()
        
        # Determine category
        category_proba = self.model.predict_proba([text])[0] if hasattr(self.model, 'predict_proba') else [0.5] * len(self.categories)
        category_idx = category_proba.argmax()
        category = self.categories[category_idx]
        confidence = float(category_proba[category_idx])
        
        # Determine priority based on keywords
        priority = 'low'
        for level, keywords in self.priority_keywords.items():
            if any(keyword in text_lower for keyword in keywords):
                priority = level
                break
        
        return {
            'category': category,
            'priority': priority,
            'confidence': confidence
        }
    
    def train(self, texts, labels):
        """Train the classifier with labeled data"""
        self.model.fit(texts, labels)
        joblib.dump(self.model, self.model_path)

ticket_classifier = TicketClassifier()
```

## Common Patterns

### Pattern 1: Protected Routes with Role-Based Access

```javascript
// Backend route protection
const express = require('express');
const router = express.Router();
const { authMiddleware, adminOnly } = require('../middleware/auth');

router.get('/users', authMiddleware, adminOnly, async (req, res) => {
  // Only admins can access
});

router.get('/tasks', authMiddleware, async (req, res) => {
  // All authenticated users can access
});
```

### Pattern 2: React Context for Authentication

```javascript
// src/context/AuthContext.jsx
import React, { createContext, useState, useContext, useEffect } from 'react';
import { authAPI } from '../services/api';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      // Verify token and get user data
      fetchCurrentUser();
    } else {
      setLoading(false);
    }
  }, []);

  const login = async (email, password) => {
    const data = await authAPI.login(email, password);
    setUser(data.user);
  };

  const logout = () => {
    authAPI.logout();
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### Pattern 3: Real-time Task Updates

```javascript
// Backend: Emit events on task updates
const io = require('socket.io')(server);

io.on('connection', (socket) => {
  socket.on('join-room', (userId) => {
    socket.join(`user-${userId}`);
  });
});

// After task update
io.to(`user-${task.assignedTo}`).emit('task-updated', task);

// Frontend: Listen for updates
import io from 'socket.io-client';

const socket = io(process.env.REACT_APP_API_URL);

useEffect(() => {
  socket.emit('join-room', user.id);
  
  socket.on('task-updated', (updatedTask) => {
    // Update local state
    setTasks(prev => prev.map(t => 
      t._id === updatedTask._id ? updatedTask : t
    ));
  });
  
  return () => socket.disconnect();
}, [user]);
```

### Pattern 4: AI-Enhanced Decision Making

```javascript
// Backend: Get AI insights before task assignment
const assignTask = async (req, res) => {
  const { taskId, userId } = req.body;
  
  // Get user workload
  const userTasks = await Task.find({ assignedTo: userId, status: { $ne: 'done' } });
  
  // Check for burnout risk
  const burnoutPrediction = await axios.post(
    `${process.env.ML_SERVICE_URL}/predict/burnout`,
    {
      user_id: userId,
      weekly_hours: calculateWeeklyHours(userTasks),
      tasks_assigned: userTasks.length
    }
  );
  
  if (burnoutPrediction.data.risk_level === 'high') {
    return res.status(400).json({
      message: 'User at high burnout risk. Consider reassigning.',
      suggestion: 'Distribute workload to other team members'
    });
  }
  
  // Proceed with assignment
  await Task.findByIdAndUpdate(taskId, { assignedTo: userId });
  res.json({ success: true });
};
```

## Configuration

### MongoDB Schema Examples

```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  failedLoginAttempts: { type: Number, default: 0 },
  lastLogin: Date,
  isActive: { type: Boolean, default: true },
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', userSchema);

// models/Task.js
const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  status: { 
    type: String, 
    enum: ['todo', 'in-progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in seconds
  delayProbability: Number,
  completedAt: Date,
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Environment Variables Reference

**Backend (.env)**:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_secure_random_secret
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**:
```env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
MAX_WORKERS=4
```

**Frontend (.env)**:
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_SOCKET_URL=http://localhost:5000
```

## Troubleshooting

### Issue: JWT Token Expired

```javascript
// Add token refresh logic
const refreshToken = async () => {
  try {
    const response = await axios.post(`${API_URL}/auth/refresh`, {}, {
      headers: getAuthHeader()
    });
    localStorage.setItem('token', response.data.token);
    return response.data.token;
  } catch (error) {
    authAPI.logout();
    window.location.href = '/login';
  }
};

// Axios interceptor for auto-refresh
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      const newToken = await refreshToken();
      error.config.headers['Authorization'] = `Bearer ${newToken}`;
      return axios.request(error.config);
    }
    return Promise.reject(error);
  }
);
```

### Issue: ML Service Connection Failure

```javascript
// Add fallback when ML service is unavailable
const getTaskInsights = async (taskData) => {
  try {
    const response = await axios.post(
      `${process.env.ML_SERVICE_URL}/predict/delay`,
      taskData,
      { timeout: 3000 }
    );
    return response.data;
  } catch (error) {
    console.warn('ML service unavailable, using fallback logic');
    // Fallback: Simple heuristic-based prediction
    return {
      delay_probability: taskData.priority === 'high' ? 0.3 : 0.5,
      confidence: 0.6,
      source: 'fallback'
    };
  }
};
```

### Issue: MongoDB Connection Issues

```javascript
// Add retry logic for MongoDB
const mongoose = require('mongoose');

const connectDB = async (retries = 5) => {
  for (let i = 0; i < retries; i++) {
    try {
      await mongoose.connect(process.env.MONGODB_URI, {
        useNewUrlParser: true,
        useUnifiedTopology: true,
        serverSelectionTimeoutMS: 5000
      });
      console.log('MongoDB connected');
      return;
    } catch (error) {
      console.error(`MongoDB connection attempt ${i + 1} failed:`, error.message);
      if (i < retries - 1) {
        await new Promise(resolve => setTimeout(resolve, 2000));
      } else {
        process.exit(1);
      }
    }
  }
};

connectDB();
```

### Issue: CORS Errors in Development

```javascript
// Backend: Configure CORS properly
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Issue: Model Files Not Found in ML Service

```python
# ml-service/models/__init__.py
import os

def ensure_model_directory():
    model_path = os.getenv('MODEL_PATH', './models')
    os.makedirs(model_path, exist_ok=True)
    
    # Create placeholder models if they don't exist
    risk_model_path = os.path.join(model_path, 'risk_model.pkl')
    if not os.path.exists(risk_model_path):
        from models.risk_predictor import RiskPredictor
        predictor = RiskPredictor(risk_model_path)
        # Train with minimal data
        predictor.train({'failed_logins': 0, 'tasks_overdue': 0}, 0)

ensure_model_directory()
```

## Deployment

### Deploy to Vercel (Frontend)

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy from frontend directory
cd frontend
vercel --prod
```

### Deploy Backend to Heroku
