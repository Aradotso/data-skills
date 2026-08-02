---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, task tracking, and intelligent ticket classification
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "show me how to implement task tracking with kanban board"
  - "configure JWT authentication for user management"
  - "set up AI ticket classification system"
  - "implement burnout detection for users"
  - "create admin dashboard with user analytics"
  - "build ML service for risk prediction"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build and use an enterprise-grade user management system with AI-powered analytics. The system includes user/admin dashboards, Kanban task tracking, support ticket management, and ML-based insights like risk detection, anomaly detection, and burnout analysis.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control, user CRUD operations, authentication
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support System**: Ticket creation, AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Admin analytics, user performance tracking, audit logs

## Architecture

The system consists of three main components:
1. **Frontend** (React.js) - User/Admin interfaces
2. **Backend** (Node.js) - REST APIs, business logic, authentication
3. **ML Service** (FastAPI) - AI/ML models for analytics

## Installation & Setup

### Prerequisites

```bash
# Required software
node >= 14.x
python >= 3.8
mongodb >= 4.x
```

### Complete Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
echo "MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
PORT=5000
ML_SERVICE_URL=http://localhost:8000" > .env
npm start

# ML Service setup (new terminal)
cd ml-service
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
echo "MODEL_PATH=./models
LOG_LEVEL=info" > .env
uvicorn main:app --reload --port 8000

# Frontend setup (new terminal)
cd frontend
npm install
echo "REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000" > .env
npm start
```

## Backend API Reference

### Authentication Endpoints

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// User Registration
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    res.status(201).json({ message: 'User registered successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// User Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    
    if (!user || !(await bcrypt.compare(password, user.password))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### Task Management Endpoints

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const authMiddleware = require('../middleware/auth');

// Create Task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, status } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      status: status || 'todo',
      createdBy: req.user.userId,
      startTime: null,
      timeSpent: 0
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update Task Status (Kanban)
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: new Date() },
      { new: true }
    );
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Time Tracking
router.post('/:id/time/start', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { startTime: new Date() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

router.post('/:id/time/stop', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    if (task.startTime) {
      const elapsed = (new Date() - task.startTime) / 1000; // seconds
      task.timeSpent += elapsed;
      task.startTime = null;
      await task.save();
    }
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### Ticket Management

```javascript
// backend/routes/tickets.js
const axios = require('axios');

// Create Ticket with AI Classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Get AI classification
    const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
      title,
      description
    });
    
    const ticket = new Ticket({
      title,
      description,
      priority: priority || mlResponse.data.priority,
      category: mlResponse.data.category,
      assignedTo: mlResponse.data.suggestedAssignee,
      createdBy: req.user.userId,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

## ML Service API

### FastAPI ML Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np
from datetime import datetime
from river import anomaly, tree

app = FastAPI()

# Models
ticket_classifier = joblib.load('models/ticket_classifier.pkl')
risk_model = joblib.load('models/risk_predictor.pkl')
anomaly_detector = anomaly.HalfSpaceTrees()

class TicketData(BaseModel):
    title: str
    description: str

class UserBehavior(BaseModel):
    userId: str
    loginTime: str
    location: str
    deviceId: str
    activityPattern: list

class TaskMetrics(BaseModel):
    userId: str
    tasksCompleted: int
    hoursWorked: float
    missedDeadlines: int
    weekNumber: int

# Ticket Classification
@app.post("/classify-ticket")
async def classify_ticket(ticket: TicketData):
    try:
        # Feature extraction
        text = f"{ticket.title} {ticket.description}"
        features = extract_ticket_features(text)
        
        # Predict category and priority
        category = ticket_classifier.predict([features])[0]
        priority = predict_priority(features)
        assignee = route_to_assignee(category)
        
        return {
            "category": category,
            "priority": priority,
            "suggestedAssignee": assignee,
            "confidence": 0.87
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Anomaly Detection
@app.post("/detect-anomaly")
async def detect_anomaly(behavior: UserBehavior):
    try:
        # Extract features
        features = {
            'hour': datetime.fromisoformat(behavior.loginTime).hour,
            'is_weekend': datetime.fromisoformat(behavior.loginTime).weekday() >= 5,
            'location_hash': hash(behavior.location) % 1000,
            'device_hash': hash(behavior.deviceId) % 1000,
            'activity_variance': np.var(behavior.activityPattern)
        }
        
        # Detect anomaly
        score = anomaly_detector.score_one(features)
        anomaly_detector.learn_one(features)
        
        is_anomaly = score > 0.7
        
        return {
            "isAnomaly": is_anomaly,
            "score": float(score),
            "riskLevel": "high" if score > 0.8 else "medium" if score > 0.5 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout Detection
@app.post("/predict-burnout")
async def predict_burnout(metrics: TaskMetrics):
    try:
        features = np.array([[
            metrics.hoursWorked,
            metrics.tasksCompleted,
            metrics.missedDeadlines,
            metrics.hoursWorked / max(metrics.tasksCompleted, 1),  # Hours per task
            metrics.weekNumber
        ]])
        
        burnout_prob = risk_model.predict_proba(features)[0][1]
        
        return {
            "burnoutRisk": float(burnout_prob),
            "riskLevel": "high" if burnout_prob > 0.7 else "medium" if burnout_prob > 0.4 else "low",
            "recommendations": generate_recommendations(burnout_prob)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def extract_ticket_features(text):
    # Simple feature extraction (replace with actual vectorizer)
    return [len(text), text.count(' '), len(set(text.split()))]

def predict_priority(features):
    # Logic for priority prediction
    if features[0] > 100:
        return "high"
    elif features[0] > 50:
        return "medium"
    return "low"

def route_to_assignee(category):
    routing = {
        "technical": "tech-team",
        "billing": "finance-team",
        "general": "support-team"
    }
    return routing.get(category, "support-team")

def generate_recommendations(burnout_prob):
    if burnout_prob > 0.7:
        return ["Reduce workload", "Schedule time off", "Delegate tasks"]
    elif burnout_prob > 0.4:
        return ["Monitor workload", "Take regular breaks"]
    return ["Maintain current pace"]
```

## Frontend Implementation

### React Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      logout();
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    
    setToken(response.data.token);
    setUser(response.data.user);
    localStorage.setItem('token', response.data.token);
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`);
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

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error moving task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable onDragStart={(e) => e.dataTransfer.setData('taskId', task._id)}>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority ${task.priority}`}>{task.priority}</span>
      {task.startTime && <TimeTracker taskId={task._id} startTime={task.startTime} />}
    </div>
  );

  const Column = ({ title, status, tasks }) => (
    <div 
      className="kanban-column"
      onDragOver={(e) => e.preventDefault()}
      onDrop={(e) => {
        const taskId = e.dataTransfer.getData('taskId');
        moveTask(taskId, status);
      }}
    >
      <h3>{title}</h3>
      <div className="task-list">
        {tasks.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <Column title="To Do" status="todo" tasks={tasks.todo} />
      <Column title="In Progress" status="in-progress" tasks={tasks.inProgress} />
      <Column title="Done" status="done" tasks={tasks.done} />
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AdminAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { LineChart, Line, BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, Legend } from 'recharts';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    burnoutAlerts: [],
    anomalies: [],
    projectInsights: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const [risks, burnouts, anomalies] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/api/analytics/risks`),
        axios.get(`${process.env.REACT_APP_API_URL}/api/analytics/burnout`),
        axios.get(`${process.env.REACT_APP_API_URL}/api/analytics/anomalies`)
      ]);

      setAnalytics({
        riskUsers: risks.data,
        burnoutAlerts: burnouts.data,
        anomalies: anomalies.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="analytics-dashboard">
      <h2>AI Analytics Dashboard</h2>
      
      <div className="alert-section">
        <h3>High Risk Users</h3>
        {analytics.riskUsers.map(user => (
          <div key={user.userId} className="alert-card risk">
            <span>{user.username}</span>
            <span>Risk Score: {(user.riskScore * 100).toFixed(1)}%</span>
            <span>Reason: {user.reason}</span>
          </div>
        ))}
      </div>

      <div className="alert-section">
        <h3>Burnout Alerts</h3>
        {analytics.burnoutAlerts.map(alert => (
          <div key={alert.userId} className="alert-card burnout">
            <span>{alert.username}</span>
            <span>Burnout Risk: {alert.riskLevel}</span>
            <ul>
              {alert.recommendations.map((rec, i) => <li key={i}>{rec}</li>)}
            </ul>
          </div>
        ))}
      </div>

      <div className="chart-section">
        <h3>Anomaly Detection Timeline</h3>
        <LineChart width={600} height={300} data={analytics.anomalies}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis dataKey="timestamp" />
          <YAxis />
          <Tooltip />
          <Legend />
          <Line type="monotone" dataKey="score" stroke="#8884d8" />
        </LineChart>
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

## Database Models

### MongoDB Schemas

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  isActive: { type: Boolean, default: true }
});

module.exports = mongoose.model('User', userSchema);

// backend/models/Task.js
const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  startTime: Date,
  timeSpent: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);

// backend/models/Ticket.js
const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: String,
  priority: { type: String, enum: ['low', 'medium', 'high', 'critical'] },
  status: { type: String, enum: ['open', 'in-progress', 'resolved', 'closed'], default: 'open' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'Authentication required' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### API Client Setup

```javascript
// frontend/src/utils/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

api.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

## Troubleshooting

### ML Service Connection Issues

```python
# ml-service/main.py - Add CORS for frontend
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### MongoDB Connection Problems

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Token Expiration Handling

```javascript
// frontend/src/utils/refreshToken.js
import api from './api';

let isRefreshing = false;
let failedQueue = [];

api.interceptors.response.use(
  response => response,
  async error => {
    const originalRequest = error.config;
    
    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        }).then(token => {
          originalRequest.headers.Authorization = `Bearer ${token}`;
          return api(originalRequest);
        });
      }
      
      originalRequest._retry = true;
      isRefreshing = true;
      
      try {
        const response = await api.post('/api/auth/refresh');
        const { token } = response.data;
        localStorage.setItem('token', token);
        
        failedQueue.forEach(prom => prom.resolve(token));
        failedQueue = [];
        
        return api(originalRequest);
      } catch (err) {
        failedQueue.forEach(prom => prom.reject(err));
        localStorage.removeItem('token');
        window.location.href = '/login';
      } finally {
        isRefreshing = false;
      }
    }
    
    return Promise.reject(error);
  }
);
```

### Environment Configuration

```bash
# backend/.env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_secure_random_string_here
JWT_EXPIRES_IN=24h
PORT=5000
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development

# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000

# ml-service/.env
MODEL_PATH=./models
LOG_LEVEL=info
ANOMALY_THRESHOLD=0.7
BURNOUT_THRESHOLD=0.7
```

This skill provides comprehensive guidance for working with the Enterprise User Management System, covering authentication, task management, AI integration, and deployment configurations.
