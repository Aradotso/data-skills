---
name: enterprise-user-management-system-ai
description: Full-stack user management system with AI-powered analytics, risk detection, and task tracking for enterprise applications
triggers:
  - "set up enterprise user management with AI"
  - "integrate AI analytics for user management"
  - "build user management system with task tracking"
  - "implement risk detection and anomaly analysis"
  - "create admin dashboard with AI insights"
  - "add burnout detection to user system"
  - "develop ticket management with AI routing"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack JavaScript/Node.js application that combines traditional user and task management with AI-powered analytics. It provides role-based access control, Kanban-style task tracking, support ticket management, and intelligent insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

**Key Components:**
- **Frontend**: React.js with JWT authentication
- **Backend**: Node.js with Express, REST APIs
- **ML Service**: FastAPI with scikit-learn and River for online learning
- **Database**: MongoDB
- **AI Features**: Ticket classification, risk prediction, anomaly detection, burnout analysis

## Installation

### Prerequisites

```bash
# Required
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

### Backend Setup

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
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
MODEL_PATH=./models
LOG_LEVEL=INFO
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
REACT_APP_ML_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Architecture

### Backend API Structure

```javascript
// server.js - Main entry point
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const dotenv = require('dotenv');

dotenv.config();
const app = express();

app.use(cors());
app.use(express.json());

// Database connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
})
.then(() => console.log('MongoDB connected'))
.catch(err => console.error('MongoDB connection error:', err));

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/tasks', require('./routes/tasks'));
app.use('/api/tickets', require('./routes/tickets'));
app.use('/api/analytics', require('./routes/analytics'));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### User Model

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please provide name'],
    trim: true
  },
  email: {
    type: String,
    required: [true, 'Please provide email'],
    unique: true,
    match: [/^\S+@\S+\.\S+$/, 'Please provide valid email']
  },
  password: {
    type: String,
    required: [true, 'Please provide password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  assignedTasks: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Task'
  }],
  activityLog: [{
    action: String,
    timestamp: Date,
    details: Object
  }],
  riskScore: {
    type: Number,
    default: 0
  },
  burnoutScore: {
    type: Number,
    default: 0
  }
}, {
  timestamps: true
});

// Hash password before save
UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) {
    next();
  }
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

// Generate JWT
UserSchema.methods.getSignedJwtToken = function() {
  return jwt.sign(
    { id: this._id, role: this.role },
    process.env.JWT_SECRET,
    { expiresIn: process.env.JWT_EXPIRE }
  );
};

// Match password
UserSchema.methods.matchPassword = async function(enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

### Task Model

```javascript
// models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  status: {
    type: String,
    enum: ['todo', 'in_progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  dueDate: {
    type: Date
  },
  timeTracking: {
    totalTime: { type: Number, default: 0 }, // in seconds
    sessions: [{
      startTime: Date,
      endTime: Date,
      duration: Number
    }]
  },
  tags: [String],
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  }
}, {
  timestamps: true
});

module.exports = mongoose.model('Task', TaskSchema);
```

### Ticket Model

```javascript
// models/Ticket.js
const mongoose = require('mongoose');

const TicketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'billing', 'general', 'urgent'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in_progress', 'resolved', 'closed'],
    default: 'open'
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  aiClassification: {
    category: String,
    confidence: Number,
    suggestedPriority: String
  },
  comments: [{
    user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
    text: String,
    timestamp: Date
  }]
}, {
  timestamps: true
});

module.exports = mongoose.model('Ticket', TicketSchema);
```

## Authentication & Authorization

### JWT Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ success: false, message: 'Not authorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id);
    next();
  } catch (error) {
    return res.status(401).json({ success: false, message: 'Token invalid' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        success: false, 
        message: `Role ${req.user.role} not authorized` 
      });
    }
    next();
  };
};
```

### Auth Routes

```javascript
// routes/auth.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');

// Register
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, department } = req.body;

    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ success: false, message: 'User already exists' });
    }

    const user = await User.create({ name, email, password, department });
    const token = user.getSignedJwtToken();

    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;

    if (!email || !password) {
      return res.status(400).json({ 
        success: false, 
        message: 'Please provide email and password' 
      });
    }

    const user = await User.findOne({ email }).select('+password');
    if (!user) {
      return res.status(401).json({ success: false, message: 'Invalid credentials' });
    }

    const isMatch = await user.matchPassword(password);
    if (!isMatch) {
      return res.status(401).json({ success: false, message: 'Invalid credentials' });
    }

    // Log activity
    user.activityLog.push({
      action: 'login',
      timestamp: new Date(),
      details: { ip: req.ip }
    });
    await user.save();

    const token = user.getSignedJwtToken();

    res.json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

## User Management Routes

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { protect, authorize } = require('../middleware/auth');

// Get all users (Admin only)
router.get('/', protect, authorize('admin'), async (req, res) => {
  try {
    const users = await User.find()
      .select('-password')
      .populate('assignedTasks', 'title status priority');
    
    res.json({ success: true, count: users.length, data: users });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Get single user
router.get('/:id', protect, async (req, res) => {
  try {
    const user = await User.findById(req.params.id)
      .select('-password')
      .populate('assignedTasks');
    
    if (!user) {
      return res.status(404).json({ success: false, message: 'User not found' });
    }

    res.json({ success: true, data: user });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Update user (Admin only)
router.put('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ success: false, message: 'User not found' });
    }

    res.json({ success: true, data: user });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Delete user (Admin only)
router.delete('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);

    if (!user) {
      return res.status(404).json({ success: false, message: 'User not found' });
    }

    res.json({ success: true, message: 'User deleted' });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

## Task Management Routes

```javascript
// routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const User = require('../models/User');
const { protect } = require('../middleware/auth');

// Get all tasks (filtered by user role)
router.get('/', protect, async (req, res) => {
  try {
    let query = {};
    
    // Regular users only see their tasks
    if (req.user.role !== 'admin') {
      query.assignedTo = req.user._id;
    }

    const tasks = await Task.find(query)
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name')
      .sort('-createdAt');

    res.json({ success: true, count: tasks.length, data: tasks });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Create task
router.post('/', protect, async (req, res) => {
  try {
    req.body.createdBy = req.user._id;
    
    const task = await Task.create(req.body);

    // Add to user's assigned tasks
    await User.findByIdAndUpdate(
      req.body.assignedTo,
      { $push: { assignedTasks: task._id } }
    );

    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status },
      { new: true, runValidators: true }
    );

    if (!task) {
      return res.status(404).json({ success: false, message: 'Task not found' });
    }

    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

// Track time
router.post('/:id/time-track', protect, async (req, res) => {
  try {
    const { startTime, endTime, duration } = req.body;
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ success: false, message: 'Task not found' });
    }

    task.timeTracking.sessions.push({ startTime, endTime, duration });
    task.timeTracking.totalTime += duration;
    await task.save();

    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
});

module.exports = router;
```

## ML Service API

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
from sklearn.preprocessing import StandardScaler
import joblib
import os
from datetime import datetime

app = FastAPI(title="Enterprise User Management AI Service")

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models directory
MODEL_PATH = os.getenv("MODEL_PATH", "./models")
os.makedirs(MODEL_PATH, exist_ok=True)

# Global models
ticket_classifier = None
anomaly_detector = None
scaler = StandardScaler()

class TicketData(BaseModel):
    title: str
    description: str
    priority: Optional[str] = "medium"

class UserActivityData(BaseModel):
    user_id: str
    login_time: str
    logout_time: str
    tasks_completed: int
    tickets_raised: int
    time_spent: float
    anomaly_score: Optional[float] = None

class BurnoutAnalysisData(BaseModel):
    user_id: str
    total_tasks: int
    completed_tasks: int
    overdue_tasks: int
    avg_daily_hours: float
    task_pressure: float

class RiskPredictionData(BaseModel):
    user_id: str
    failed_login_attempts: int
    unusual_access_times: int
    data_download_volume: float
    permission_changes: int
    activity_score: float

@app.on_event("startup")
async def load_models():
    """Load or initialize ML models"""
    global ticket_classifier, anomaly_detector
    
    try:
        ticket_classifier = joblib.load(f"{MODEL_PATH}/ticket_classifier.pkl")
        anomaly_detector = joblib.load(f"{MODEL_PATH}/anomaly_detector.pkl")
    except:
        # Initialize new models
        ticket_classifier = RandomForestClassifier(n_estimators=100, random_state=42)
        anomaly_detector = IsolationForest(contamination=0.1, random_state=42)

@app.get("/")
async def root():
    return {"message": "Enterprise User Management AI Service", "status": "active"}

@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketData):
    """Classify ticket category and priority using NLP"""
    try:
        # Simple keyword-based classification
        text = f"{ticket.title} {ticket.description}".lower()
        
        # Category detection
        if any(word in text for word in ['bug', 'error', 'crash', 'issue']):
            category = 'technical'
            confidence = 0.85
        elif any(word in text for word in ['payment', 'invoice', 'billing', 'refund']):
            category = 'billing'
            confidence = 0.82
        elif any(word in text for word in ['urgent', 'critical', 'emergency', 'asap']):
            category = 'urgent'
            confidence = 0.90
        else:
            category = 'general'
            confidence = 0.70
        
        # Priority suggestion
        if 'urgent' in text or 'critical' in text:
            suggested_priority = 'high'
        elif 'soon' in text or 'important' in text:
            suggested_priority = 'medium'
        else:
            suggested_priority = 'low'
        
        return {
            "success": True,
            "category": category,
            "confidence": confidence,
            "suggested_priority": suggested_priority
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(activity: UserActivityData):
    """Detect anomalous user behavior"""
    try:
        # Extract features
        features = np.array([[
            activity.tasks_completed,
            activity.tickets_raised,
            activity.time_spent
        ]])
        
        # Fit and predict (in production, use pre-trained model)
        if features.shape[0] > 0:
            prediction = anomaly_detector.fit_predict(features)
            anomaly_score = anomaly_detector.score_samples(features)[0]
            
            is_anomaly = prediction[0] == -1
            risk_level = "high" if is_anomaly else "normal"
            
            return {
                "success": True,
                "is_anomaly": bool(is_anomaly),
                "anomaly_score": float(anomaly_score),
                "risk_level": risk_level
            }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/analyze-burnout")
async def analyze_burnout(data: BurnoutAnalysisData):
    """Analyze user burnout risk"""
    try:
        # Calculate burnout score (0-100)
        overdue_ratio = data.overdue_tasks / max(data.total_tasks, 1)
        completion_ratio = data.completed_tasks / max(data.total_tasks, 1)
        
        burnout_score = (
            (overdue_ratio * 30) +
            ((1 - completion_ratio) * 25) +
            (min(data.avg_daily_hours / 12, 1) * 25) +
            (data.task_pressure * 20)
        )
        
        burnout_score = min(burnout_score, 100)
        
        if burnout_score > 70:
            risk_level = "high"
            recommendation = "Immediate workload reduction recommended"
        elif burnout_score > 40:
            risk_level = "medium"
            recommendation = "Monitor workload and consider task redistribution"
        else:
            risk_level = "low"
            recommendation = "Workload is manageable"
        
        return {
            "success": True,
            "burnout_score": round(burnout_score, 2),
            "risk_level": risk_level,
            "recommendation": recommendation,
            "metrics": {
                "overdue_ratio": round(overdue_ratio, 2),
                "completion_ratio": round(completion_ratio, 2),
                "avg_daily_hours": data.avg_daily_hours
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-risk")
async def predict_risk(data: RiskPredictionData):
    """Predict security risk for user"""
    try:
        # Calculate risk score (0-100)
        risk_score = (
            (data.failed_login_attempts * 15) +
            (data.unusual_access_times * 20) +
            (min(data.data_download_volume / 1000, 1) * 25) +
            (data.permission_changes * 20) +
            ((1 - data.activity_score) * 20)
        )
        
        risk_score = min(risk_score, 100)
        
        if risk_score > 70:
            risk_level = "critical"
            action = "Immediate security review required"
        elif risk_score > 40:
            risk_level = "medium"
            action = "Monitor user activity closely"
        else:
            risk_level = "low"
            action = "No immediate action required"
        
        return {
            "success": True,
            "risk_score": round(risk_score, 2),
            "risk_level": risk_level,
            "recommended_action": action,
            "factors": {
                "failed_logins": data.failed_login_attempts,
                "unusual_access": data.unusual_access_times,
                "data_volume": data.data_download_volume,
                "permission_changes": data.permission_changes
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Frontend Integration

### API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

// Create axios instance with auth
const api = axios.create({
  baseURL: API_URL
});

// Add token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auth
export const login = (credentials) => api.post('/api/auth/login', credentials);
export const register = (userData) => api.post('/api/auth/register', userData);

// Users
export const getUsers = () => api.get('/api/users');
export const getUser = (id) => api.get(`/api/users/${id}`);
export const updateUser = (id, data) => api.put(`/api/users/${id}`, data);
export const deleteUser = (id) => api.delete(`/api/users/${id}`);

// Tasks
export const getTasks = () => api.get('/api/tasks');
export const createTask = (taskData) => api.post('/api/tasks', taskData);
export const updateTaskStatus = (id, status) => 
  api.patch(`/api/tasks/${id}/status`, { status });
export const trackTime = (id, timeData) => 
  api.post(`/api/tasks/${id}/time-track`, timeData);

// Tickets
export const getTickets = () => api.get('/api/tickets');
export const createTicket = (ticketData) => api.post('/api/tickets', ticketData);

// ML Services
export const classifyTicket = (ticketData) => 
  axios.post(`${ML_URL}/api/ml/classify-ticket`, ticketData);
export const detectAnomaly = (activityData) => 
  axios.post(`${ML_URL}/api/ml/detect-anomaly`, activityData);
export const analyzeBurnout = (burnoutData) => 
  axios.post(`${ML_URL}/api/ml/analyze-burnout`, burnoutData);
export const predictRisk = (riskData) => 
  axios.post(`${ML_URL}/api/ml/predict-risk`, riskData);

export default api;
```

### React Dashboard Component

```javascript
// frontend/src/components/Dashboard.jsx
import React, { useState, useEffect } from 'react';
import { getTasks, updateTaskStatus, analyzeBurnout } from '../services/api';
import './Dashboard.css';

const Dashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const tasksRes = await getTasks();
      setTasks(tasksRes.data.data);

      // Analyze burnout
      const totalTasks = tasksRes.data.count;
      const completed = tasksRes.data.data.filter(t => t.status === 'done').length;
      const overdue = tasksRes.data.data.filter(t => 
        new Date(t.dueDate) < new Date() && t.status !== 'done'
      ).length;

      const burnoutRes = await analyzeBurnout({
        user_id: localStorage.getItem('userId'),
        total_tasks: totalTasks,
        completed_tasks: completed,
        overdue_tasks: overdue,
        avg_daily_hours: 8,
        task_pressure: 0.6
      });

      setBurnoutData(burnoutRes.data);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
      setLoading(false);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await updateTaskStatus(taskId, newStatus);
      fetchDashboardData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const getTasksByStatus = (status) => {
    return tasks.filter(task => task.status === status);
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <header className="dashboard-header">
        <h1>My Dashboard</h1>
        {burnoutData && (
          <div className={`burnout-indicator ${burnoutData.risk_level}`}>
            <span>Burnout Risk: {burnoutData.risk_level}</span>
            <span>{burnoutData.burnout_score}%</span>
          </div>
        )}
      </header>

      <div className="kanban-board">
        <div className="kanban-column">
          <h3>To Do ({getTasksByStatus('todo').length})</h3>
          {getTasksByStatus('todo').map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={handleStatusChange}
            />
          ))}
        </div>

        <div className="kanban-column">
          <h3>In Progress ({getTasksByStatus('in_progress').length})</h3>
          {getTasksBy
