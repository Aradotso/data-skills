---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management capabilities
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user task tracking with AI insights"
  - "build admin dashboard with anomaly detection"
  - "integrate AI-powered ticket classification"
  - "add burnout detection to user management"
  - "deploy enterprise user management with ML"
  - "configure AI risk prediction for users"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What It Does

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user management, task tracking, and support ticket systems with AI-powered insights. The system provides:

- **User & Admin Management**: Role-based access control with JWT authentication
- **Task Management**: Kanban board with time tracking and progress monitoring
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Real-time Insights**: Dashboard analytics with audit logs and alerts

The stack consists of:
- **Frontend**: React.js
- **Backend**: Node.js with REST APIs
- **ML Service**: FastAPI with scikit-learn and River
- **Database**: MongoDB

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
python >= 3.8
mongodb >= 4.x
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

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

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=${MONGODB_URI}
MODEL_PATH=./models
LOG_LEVEL=info
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Project Structure

```
enterprise-user-management/
├── frontend/           # React application
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── services/
│   │   └── utils/
├── backend/           # Node.js REST API
│   ├── models/
│   ├── routes/
│   ├── controllers/
│   ├── middleware/
│   └── server.js
└── ml-service/        # FastAPI ML service
    ├── models/
    ├── services/
    ├── routes/
    └── main.py
```

## Backend API Patterns

### User Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

const router = express.Router();

// Register user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate JWT
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );
    
    res.status(201).json({ token, user: { id: user._id, username, email, role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const Task = require('../models/Task');
const { authMiddleware } = require('../middleware/auth');

const router = express.Router();

// Get user tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority, dueDate, assignedTo } = req.body;
    
    const task = new Task({
      title,
      description,
      priority,
      dueDate,
      status: 'todo',
      assignedTo: assignedTo || req.user.id,
      createdBy: req.user.id
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body; // in seconds
    
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { $inc: { timeSpent: duration } },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Tickets

```javascript
// backend/routes/tickets.js
const express = require('express');
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authMiddleware } = require('../middleware/auth');

const router = express.Router();

// Create ticket with AI classification
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
      priority,
      category: mlResponse.data.category,
      assignedDepartment: mlResponse.data.department,
      createdBy: req.user.id,
      status: 'open'
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tickets
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service API

### AI Ticket Classification

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import joblib
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB

app = FastAPI()

# Load or initialize models
try:
    ticket_classifier = joblib.load('./models/ticket_classifier.pkl')
    vectorizer = joblib.load('./models/vectorizer.pkl')
except:
    vectorizer = TfidfVectorizer(max_features=1000)
    ticket_classifier = MultinomialNB()

class TicketInput(BaseModel):
    title: str
    description: str

class TicketClassification(BaseModel):
    category: str
    department: str
    confidence: float

@app.post("/classify-ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketInput):
    try:
        # Combine title and description
        text = f"{ticket.title} {ticket.description}"
        
        # Vectorize
        features = vectorizer.transform([text])
        
        # Predict
        category_pred = ticket_classifier.predict(features)[0]
        confidence = ticket_classifier.predict_proba(features).max()
        
        # Map category to department
        department_mapping = {
            'technical': 'IT',
            'billing': 'Finance',
            'general': 'Support',
            'urgent': 'Priority'
        }
        
        return TicketClassification(
            category=category_pred,
            department=department_mapping.get(category_pred, 'Support'),
            confidence=float(confidence)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Risk Prediction

```python
# ml-service/services/risk_prediction.py
from pydantic import BaseModel
from typing import List
import numpy as np
from river import anomaly, preprocessing

class UserActivity(BaseModel):
    user_id: str
    login_count: int
    failed_logins: int
    tasks_completed: int
    tasks_overdue: int
    time_spent_hours: float
    ticket_count: int

class RiskScore(BaseModel):
    user_id: str
    risk_level: str
    score: float
    factors: List[str]

# Initialize online learning model
scaler = preprocessing.StandardScaler()
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)

@app.post("/predict-risk", response_model=RiskScore)
async def predict_risk(activity: UserActivity):
    try:
        # Create feature vector
        features = {
            'login_count': activity.login_count,
            'failed_logins': activity.failed_logins,
            'tasks_completed': activity.tasks_completed,
            'tasks_overdue': activity.tasks_overdue,
            'time_spent_hours': activity.time_spent_hours,
            'ticket_count': activity.ticket_count,
            'overdue_ratio': activity.tasks_overdue / max(activity.tasks_completed, 1)
        }
        
        # Scale features
        scaled_features = scaler.transform_one(features)
        
        # Get anomaly score
        score = anomaly_detector.score_one(scaled_features)
        anomaly_detector.learn_one(scaled_features)
        
        # Determine risk level
        risk_level = 'low'
        factors = []
        
        if score > 0.7:
            risk_level = 'high'
        elif score > 0.4:
            risk_level = 'medium'
        
        # Identify risk factors
        if activity.failed_logins > 5:
            factors.append('High failed login attempts')
        if activity.tasks_overdue > 3:
            factors.append('Multiple overdue tasks')
        if activity.time_spent_hours < 10:
            factors.append('Low activity time')
        
        return RiskScore(
            user_id=activity.user_id,
            risk_level=risk_level,
            score=float(score),
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Burnout Detection

```python
# ml-service/services/burnout_detection.py
from pydantic import BaseModel
from typing import Dict

class WorkloadData(BaseModel):
    user_id: str
    active_tasks: int
    completed_last_week: int
    avg_hours_per_day: float
    weekend_work_hours: float
    consecutive_work_days: int

class BurnoutAnalysis(BaseModel):
    user_id: str
    burnout_risk: str
    score: float
    recommendations: List[str]

@app.post("/detect-burnout", response_model=BurnoutAnalysis)
async def detect_burnout(workload: WorkloadData):
    try:
        score = 0.0
        recommendations = []
        
        # Calculate burnout score
        if workload.avg_hours_per_day > 10:
            score += 0.3
            recommendations.append("Reduce daily working hours")
        
        if workload.weekend_work_hours > 8:
            score += 0.25
            recommendations.append("Minimize weekend work")
        
        if workload.consecutive_work_days > 10:
            score += 0.2
            recommendations.append("Take time off")
        
        if workload.active_tasks > 15:
            score += 0.15
            recommendations.append("Delegate some tasks")
        
        task_completion_rate = workload.completed_last_week / max(workload.active_tasks, 1)
        if task_completion_rate < 0.3:
            score += 0.1
            recommendations.append("Review task priorities")
        
        # Determine risk level
        if score > 0.7:
            risk = 'high'
        elif score > 0.4:
            risk = 'medium'
        else:
            risk = 'low'
            recommendations = ["Maintain current work-life balance"]
        
        return BurnoutAnalysis(
            user_id=workload.user_id,
            burnout_risk=risk,
            score=score,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend Integration

### API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

// Create axios instance
const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auth API
export const authAPI = {
  login: (credentials) => api.post('/api/auth/login', credentials),
  register: (userData) => api.post('/api/auth/register', userData),
  getCurrentUser: () => api.get('/api/auth/me')
};

// Task API
export const taskAPI = {
  getTasks: () => api.get('/api/tasks'),
  createTask: (taskData) => api.post('/api/tasks', taskData),
  updateTaskStatus: (id, status) => api.patch(`/api/tasks/${id}/status`, { status }),
  trackTime: (id, duration) => api.post(`/api/tasks/${id}/time`, { duration })
};

// Ticket API
export const ticketAPI = {
  getTickets: () => api.get('/api/tickets'),
  createTicket: (ticketData) => api.post('/api/tickets', ticketData),
  updateTicket: (id, updates) => api.patch(`/api/tickets/${id}`, updates)
};

// AI Analytics API
export const analyticsAPI = {
  getRiskScore: (userId, activityData) => 
    axios.post(`${ML_API_URL}/predict-risk`, activityData),
  detectBurnout: (userId, workloadData) => 
    axios.post(`${ML_API_URL}/detect-burnout`, workloadData),
  classifyTicket: (ticketData) => 
    axios.post(`${ML_API_URL}/classify-ticket`, ticketData)
};

export default api;
```

### Task Board Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';
import './TaskBoard.css';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskAPI.getTasks();
      const groupedTasks = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(groupedTasks);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskAPI.updateTaskStatus(taskId, newStatus);
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const renderColumn = (status, title, taskList) => (
    <div className="kanban-column">
      <h3>{title} ({taskList.length})</h3>
      <div className="task-list">
        {taskList.map(task => (
          <div key={task._id} className={`task-card priority-${task.priority}`}>
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <div className="task-meta">
              <span className="priority-badge">{task.priority}</span>
              <span className="time-spent">{Math.round(task.timeSpent / 60)}m</span>
            </div>
            <select 
              value={task.status} 
              onChange={(e) => handleStatusChange(task._id, e.target.value)}
            >
              <option value="todo">To Do</option>
              <option value="in-progress">In Progress</option>
              <option value="done">Done</option>
            </select>
          </div>
        ))}
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do', tasks.todo)}
      {renderColumn('in-progress', 'In Progress', tasks.inProgress)}
      {renderColumn('done', 'Done', tasks.done)}
    </div>
  );
};

export default TaskBoard;
```

### Admin Analytics Dashboard

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { analyticsAPI } from '../services/api';
import './AdminDashboard.css';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    totalUsers: 0,
    activeTasks: 0,
    openTickets: 0,
    riskAlerts: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch user risk scores
      const users = await api.get('/api/admin/users');
      
      const riskPromises = users.data.map(async (user) => {
        const activityData = {
          user_id: user._id,
          login_count: user.loginCount || 0,
          failed_logins: user.failedLogins || 0,
          tasks_completed: user.tasksCompleted || 0,
          tasks_overdue: user.tasksOverdue || 0,
          time_spent_hours: user.timeSpentHours || 0,
          ticket_count: user.ticketCount || 0
        };
        
        const riskResponse = await analyticsAPI.getRiskScore(user._id, activityData);
        return { user: user.username, ...riskResponse.data };
      });
      
      const risks = await Promise.all(riskPromises);
      const highRisks = risks.filter(r => r.risk_level === 'high');
      
      setAnalytics(prev => ({
        ...prev,
        riskAlerts: highRisks
      }));
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h2>Admin Analytics Dashboard</h2>
      
      <div className="metrics-grid">
        <div className="metric-card">
          <h3>Total Users</h3>
          <p className="metric-value">{analytics.totalUsers}</p>
        </div>
        <div className="metric-card">
          <h3>Active Tasks</h3>
          <p className="metric-value">{analytics.activeTasks}</p>
        </div>
        <div className="metric-card">
          <h3>Open Tickets</h3>
          <p className="metric-value">{analytics.openTickets}</p>
        </div>
      </div>

      <div className="risk-alerts">
        <h3>High Risk Users</h3>
        {analytics.riskAlerts.length === 0 ? (
          <p>No high-risk users detected</p>
        ) : (
          <ul>
            {analytics.riskAlerts.map((alert, idx) => (
              <li key={idx} className="alert-item">
                <strong>{alert.user}</strong>
                <span className="risk-score">Risk: {(alert.score * 100).toFixed(0)}%</span>
                <ul className="risk-factors">
                  {alert.factors.map((factor, i) => (
                    <li key={i}>{factor}</li>
                  ))}
                </ul>
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Configuration

### Environment Variables

**Backend (.env)**:
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your-secret-key-min-32-chars
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**:
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=info
MAX_WORKERS=4
```

**Frontend (.env)**:
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

### MongoDB Models

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  loginCount: { type: Number, default: 0 },
  failedLogins: { type: Number, default: 0 },
  lastLogin: { type: Date },
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', userSchema);
```

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
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
  timeSpent: { type: Number, default: 0 }, // in seconds
  dueDate: { type: Date },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

## Common Patterns

### Protected Routes

```javascript
// frontend/src/App.jsx
import React from 'react';
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import Login from './pages/Login';
import Dashboard from './pages/Dashboard';
import AdminDashboard from './pages/AdminDashboard';

const PrivateRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route 
          path="/dashboard" 
          element={
            <PrivateRoute>
              <Dashboard />
            </PrivateRoute>
          } 
        />
        <Route 
          path="/admin" 
          element={
            <PrivateRoute adminOnly>
              <AdminDashboard />
            </PrivateRoute>
          } 
        />
        <Route path="/" element={<Navigate to="/dashboard" />} />
      </Routes>
    </BrowserRouter>
  );
}

export default App;
```

### Time Tracking Hook

```javascript
// frontend/src/hooks/useTimeTracker.js
import { useState, useEffect, useRef } from 'react';
import { taskAPI } from '../services/api';

export const useTimeTracker = (taskId) => {
  const [isTracking, setIsTracking] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (isTracking) {
      intervalRef.current = setInterval(() => {
        setElapsedTime(prev => prev + 1);
      }, 1000);
    } else {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    }

    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    };
  }, [isTracking]);

  const start = () => setIsTracking(true);
  
  const stop = async () => {
    setIsTracking(false);
    if (elapsedTime > 0) {
      try {
        await taskAPI.trackTime(taskId, elapsedTime);
      } catch (error) {
        console.error('Error tracking time:', error);
      }
    }
  };

  const reset = () => {
    setElapsedTime(0);
    setIsTracking(false);
  };

  return { isTracking, elapsedTime, start, stop, reset };
};
```

### AI-Powered Ticket Creation

```javascript
// frontend/src/components/CreateTicket.jsx
import React, { useState } from 'react';
import { ticketAPI, analyticsAPI } from '../services/api';

const CreateTicket = () => {
