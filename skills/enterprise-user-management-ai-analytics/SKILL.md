---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user management"
  - "create task management with AI analytics"
  - "build user dashboard with burnout detection"
  - "integrate AI ticket classification system"
  - "add risk prediction to user management"
  - "configure enterprise task tracking system"
  - "deploy user management with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack application for managing enterprise users, tasks, and support tickets with integrated AI/ML capabilities for risk detection, anomaly detection, burnout analysis, and predictive project insights.

## What It Does

- **User Management**: JWT-based authentication, role-based access control (Admin/User)
- **Task Management**: Kanban-style boards (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-powered classification and automated routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay forecasting
- **Admin Dashboard**: Organization-wide analytics, audit logs, user monitoring
- **ML Service**: FastAPI-based microservice with scikit-learn and River for online learning

## Installation

### Prerequisites

```bash
# Required tools
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
```

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup (Node.js)

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
```

### ML Service Setup (Python/FastAPI)

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
API_HOST=0.0.0.0
API_PORT=8000
EOF

uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
EOF

npm start
```

## Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   React     │─────▶│   Node.js   │─────▶│   MongoDB   │
│  Frontend   │      │   Backend   │      │  Database   │
└─────────────┘      └──────┬──────┘      └─────────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │   FastAPI   │
                     │  ML Service │
                     └─────────────┘
```

## Backend API Usage

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

const router = express.Router();

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });

    await user.save();
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );

    res.status(201).json({ token, user: { id: user._id, username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    const isValidPassword = await bcrypt.compare(password, user.password);
    if (!isValidPassword) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );

    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { authenticate } = require('../middleware/auth');

const router = express.Router();

// Get all tasks for user
router.get('/', authenticate, async (req, res) => {
  try {
    const tasks = await Task.find({ 
      $or: [
        { assignedTo: req.user.userId },
        { createdBy: req.user.userId }
      ]
    }).populate('assignedTo', 'username email');
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching tasks', error: error.message });
  }
});

// Create task
router.post('/', authenticate, async (req, res) => {
  try {
    const { title, description, priority, dueDate, assignedTo } = req.body;
    
    const task = new Task({
      title,
      description,
      priority,
      dueDate,
      assignedTo,
      createdBy: req.user.userId,
      status: 'todo'
    });

    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error creating task', error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authenticate, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error updating task', error: error.message });
  }
});

module.exports = router;
```

### Ticket Management API

```javascript
// backend/routes/tickets.js
const express = require('express');
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authenticate } = require('../middleware/auth');

const router = express.Router();

// Create ticket with AI classification
router.post('/', authenticate, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for ticket classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
      title,
      description
    });

    const { category, suggestedAssignee, urgencyScore } = mlResponse.data;

    const ticket = new Ticket({
      title,
      description,
      priority: priority || 'medium',
      category,
      suggestedAssignee,
      urgencyScore,
      createdBy: req.user.userId,
      status: 'open'
    });

    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: 'Error creating ticket', error: error.message });
  }
});

// Get all tickets
router.get('/', authenticate, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.userId };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching tickets', error: error.message });
  }
});

module.exports = router;
```

## ML Service API

### FastAPI Main Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, linear_model
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
ticket_classifier = None
risk_predictor = None
anomaly_detector = anomaly.HalfSpaceTrees(seed=42)

class TicketRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    userId: str
    loginFrequency: int
    failedLogins: int
    taskCompletionRate: float
    avgResponseTime: float

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    tasksInProgress: int
    avgTaskDuration: float
    overtimeHours: float
    weeklyWorkload: float

class AnomalyDetectionRequest(BaseModel):
    userId: str
    features: List[float]

@app.on_event("startup")
async def load_models():
    global ticket_classifier, risk_predictor
    
    model_path = os.getenv('MODEL_PATH', './models')
    
    # Load or initialize models
    try:
        ticket_classifier = joblib.load(f'{model_path}/ticket_classifier.pkl')
    except:
        # Initialize with dummy model
        ticket_classifier = RandomForestClassifier(n_estimators=100)
    
    try:
        risk_predictor = joblib.load(f'{model_path}/risk_predictor.pkl')
    except:
        risk_predictor = RandomForestClassifier(n_estimators=100)

@app.post("/classify-ticket")
async def classify_ticket(request: TicketRequest):
    """AI-powered ticket classification and routing"""
    try:
        # Simple rule-based classification (replace with trained model)
        text = f"{request.title} {request.description}".lower()
        
        if any(word in text for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
            urgency = 0.8
        elif any(word in text for word in ['access', 'permission', 'login']):
            category = 'security'
            urgency = 0.9
        elif any(word in text for word in ['feature', 'enhancement', 'suggestion']):
            category = 'feature_request'
            urgency = 0.3
        else:
            category = 'general'
            urgency = 0.5
        
        return {
            "category": category,
            "suggestedAssignee": "admin",
            "urgencyScore": urgency
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavioral patterns"""
    try:
        features = np.array([[
            request.loginFrequency,
            request.failedLogins,
            request.taskCompletionRate,
            request.avgResponseTime
        ]])
        
        # Simple risk scoring logic
        risk_score = 0.0
        
        if request.failedLogins > 5:
            risk_score += 0.3
        if request.taskCompletionRate < 0.5:
            risk_score += 0.2
        if request.avgResponseTime > 24:  # hours
            risk_score += 0.2
        if request.loginFrequency > 50:  # per day
            risk_score += 0.3
        
        risk_level = "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        
        return {
            "userId": request.userId,
            "riskScore": min(risk_score, 1.0),
            "riskLevel": risk_level,
            "factors": {
                "failedLogins": request.failedLogins,
                "taskCompletionRate": request.taskCompletionRate
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutAnalysisRequest):
    """Analyze workload and predict burnout risk"""
    try:
        burnout_score = 0.0
        
        if request.tasksInProgress > 10:
            burnout_score += 0.3
        if request.overtimeHours > 20:
            burnout_score += 0.4
        if request.weeklyWorkload > 60:
            burnout_score += 0.3
        
        burnout_level = "critical" if burnout_score > 0.7 else "warning" if burnout_score > 0.4 else "normal"
        
        recommendations = []
        if request.tasksInProgress > 10:
            recommendations.append("Redistribute tasks to other team members")
        if request.overtimeHours > 20:
            recommendations.append("Reduce overtime hours immediately")
        if request.weeklyWorkload > 60:
            recommendations.append("Schedule time off or reduce workload")
        
        return {
            "userId": request.userId,
            "burnoutScore": min(burnout_score, 1.0),
            "burnoutLevel": burnout_level,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Real-time anomaly detection using online learning"""
    try:
        features_dict = {f"f{i}": v for i, v in enumerate(request.features)}
        
        # Get anomaly score
        score = anomaly_detector.score_one(features_dict)
        
        # Update model with new data point
        anomaly_detector.learn_one(features_dict)
        
        is_anomaly = score > 0.6  # Threshold
        
        return {
            "userId": request.userId,
            "anomalyScore": float(score),
            "isAnomaly": is_anomaly,
            "severity": "high" if score > 0.8 else "medium" if score > 0.6 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Frontend Integration

### Authentication Service

```javascript
// frontend/src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const authService = {
  async login(email, password) {
    const response = await axios.post(`${API_URL}/auth/login`, {
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    
    return response.data;
  },

  async register(userData) {
    const response = await axios.post(`${API_URL}/auth/register`, userData);
    return response.data;
  },

  logout() {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  },

  getCurrentUser() {
    const user = localStorage.getItem('user');
    return user ? JSON.parse(user) : null;
  },

  getToken() {
    return localStorage.getItem('token');
  }
};
```

### API Client with Authentication

```javascript
// frontend/src/services/apiClient.js
import axios from 'axios';
import { authService } from './authService';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

// Add auth token to requests
apiClient.interceptors.request.use(
  (config) => {
    const token = authService.getToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Handle auth errors
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      authService.logout();
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

### Task Management Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import apiClient from '../services/apiClient';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await apiClient.get('/tasks');
      const tasksByStatus = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'inProgress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await apiClient.patch(`/tasks/${taskId}/status`, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority ${task.priority}`}>{task.priority}</span>
    </div>
  );

  if (loading) return <div>Loading...</div>;

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      <div className="kanban-column">
        <h3>Done</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import apiClient from '../services/apiClient';

const ML_URL = process.env.REACT_APP_ML_URL;

const AIAnalytics = ({ userId }) => {
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch user stats from backend
      const userStats = await apiClient.get(`/users/${userId}/stats`);
      
      // Get risk prediction
      const riskResponse = await axios.post(`${ML_URL}/predict-risk`, {
        userId,
        loginFrequency: userStats.data.loginFrequency,
        failedLogins: userStats.data.failedLogins,
        taskCompletionRate: userStats.data.taskCompletionRate,
        avgResponseTime: userStats.data.avgResponseTime
      });
      setRiskData(riskResponse.data);

      // Get burnout analysis
      const burnoutResponse = await axios.post(`${ML_URL}/detect-burnout`, {
        userId,
        tasksInProgress: userStats.data.tasksInProgress,
        avgTaskDuration: userStats.data.avgTaskDuration,
        overtimeHours: userStats.data.overtimeHours,
        weeklyWorkload: userStats.data.weeklyWorkload
      });
      setBurnoutData(burnoutResponse.data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="analytics-card">
        <h3>Risk Assessment</h3>
        <div className={`risk-level ${riskData?.riskLevel}`}>
          {riskData?.riskLevel.toUpperCase()}
        </div>
        <p>Risk Score: {(riskData?.riskScore * 100).toFixed(0)}%</p>
      </div>

      <div className="analytics-card">
        <h3>Burnout Analysis</h3>
        <div className={`burnout-level ${burnoutData?.burnoutLevel}`}>
          {burnoutData?.burnoutLevel.toUpperCase()}
        </div>
        <p>Burnout Score: {(burnoutData?.burnoutScore * 100).toFixed(0)}%</p>
        {burnoutData?.recommendations.length > 0 && (
          <div className="recommendations">
            <h4>Recommendations:</h4>
            <ul>
              {burnoutData.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        )}
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: {
    type: Date
  },
  isActive: {
    type: Boolean,
    default: true
  }
});

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  status: {
    type: String,
    enum: ['todo', 'inProgress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
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
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', taskSchema);
```

## Middleware

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authenticate = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authenticate, requireAdmin };
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your-secure-jwt-secret-key-here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**
```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
API_HOST=0.0.0.0
API_PORT=8000
PYTHONUNBUFFERED=1
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

## Common Patterns

### Creating a New Feature Module

1. **Define the Model**
```javascript
// backend/models/Department.js
const mongoose = require('mongoose');

const departmentSchema = new mongoose.Schema({
  name: String,
  manager: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  members: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }]
});

module.exports = mongoose.model('Department', departmentSchema);
```

2. **Create Routes**
```javascript
// backend/routes/departments.js
const express = require('express');
const Department = require('../models/Department');
const { authenticate, requireAdmin } = require('../middleware/auth');

const router = express.Router();

router.get('/', authenticate, async (req, res) => {
  const departments = await Department.find().populate('manager members');
  res.json(departments);
});

router.post('/', authenticate, requireAdmin, async (req, res) => {
  const dept = new Department(req.body);
  await dept.save();
  res.status(201).json(dept);
});

module.exports = router;
```

3. **Add to Main App**
```javascript
// backend/app.js
const departmentRoutes = require('./routes/departments');
app.use('/api/departments', departmentRoutes);
```

### Adding ML Endpoint

```python
# ml-service/main.py
from pydantic import BaseModel

class PredictionRequest(BaseModel):
    features: List[float]

@app.post("/predict-performance")
async def predict_performance(request: PredictionRequest):
    # Your ML logic
    prediction = model.predict([request.features])
    return {"prediction": float(prediction[0])}
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Issues

```javascript
// backend/app.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Model Loading

```python
# ml-service/main.py
import os
from pathlib import Path

@app.on_event("startup")
async def load_models():
    model_path = Path(os.getenv('MODEL_PATH', './models'))
    model_path.mkdir(parents=True, exist_ok=True)
    
    try:
        global ticket_classifier
        ticket_classifier = joblib.load(model_path / 'ticket_classifier.pkl')
    except FileNotFoundError:
        print("Model not found, initializing new model")
        ticket_classifier = RandomForestClassifier()
```

### Token Expiration Handling

```javascript
// frontend/src/services/apiClient.js
apiClient.interceptors.response.use(
  response => response,
  async error => {
    const originalRequest = error.config;
    
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      authService.logout();
      window.location.href = '/login';
    }
    
    return Promise.reject(error
