---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with ML"
  - "build task management with AI insights"
  - "integrate AI ticket classification system"
  - "deploy user management with anomaly detection"
  - "configure enterprise task tracking system"
  - "implement burnout detection for users"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with machine learning capabilities. It provides role-based access control, Kanban-style task tracking, support ticket management, and AI-driven insights including risk prediction, anomaly detection, burnout analysis, and predictive project analytics.

**Core Components:**
- **Frontend**: React.js-based admin and user dashboards
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based AI analytics using scikit-learn and River
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

```bash
# Ensure you have installed
node -v  # v14+ required
npm -v
python --version  # Python 3.8+ required
pip --version
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
```

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
# or for development
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
BACKEND_URL=http://localhost:5000
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

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

Access the application at `http://localhost:3000`

## Architecture

### Backend API Structure

```javascript
// backend/server.js - Main server setup
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const jwt = require('jsonwebtoken');
require('dotenv').config();

const app = express();

app.use(cors());
app.use(express.json());

// MongoDB connection
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
});

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
  console.log(`Backend server running on port ${PORT}`);
});
```

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
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
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  lastLogin: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

module.exports = mongoose.model('User', userSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      throw new Error('No token provided');
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');
    
    if (!user || user.status !== 'active') {
      throw new Error('User not found or inactive');
    }
    
    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
  }
};

// Admin-only middleware
const adminAuth = async (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { auth, adminAuth };
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
  description: String,
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  dueDate: Date,
  timeSpent: {
    type: Number,
    default: 0 // in minutes
  },
  tags: [String],
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### API Routes

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'Email already registered' });
    }
    
    const user = new User({ name, email, password, role, department });
    await user.save();
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      },
      token
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Login
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
    
    if (user.status !== 'active') {
      return res.status(403).json({ error: 'Account is not active' });
    }
    
    user.lastLogin = new Date();
    await user.save();
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      },
      token
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { auth } = require('../middleware/auth');

// Get user tasks
router.get('/my-tasks', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user._id })
      .populate('assignedBy', 'name email')
      .sort({ createdAt: -1 });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', auth, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      assignedBy: req.user._id
    });
    await task.save();
    await task.populate('assignedTo assignedBy', 'name email');
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    if (task.assignedTo.toString() !== req.user._id.toString()) {
      return res.status(403).json({ error: 'Not authorized' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time
router.patch('/:id/time', auth, async (req, res) => {
  try {
    const { minutes } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeSpent: minutes } },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Ticket Model and Routes

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'account', 'billing', 'general'],
    required: true
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
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
    priority: String,
    confidence: Number
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { auth, adminAuth } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', auth, async (req, res) => {
  try {
    const { title, description, category } = req.body;
    
    // Get AI classification
    let aiClassification = null;
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        title,
        description
      });
      aiClassification = mlResponse.data;
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      category: aiClassification?.category || category,
      priority: aiClassification?.priority || 'medium',
      createdBy: req.user._id,
      aiClassification
    });
    
    await ticket.save();
    await ticket.populate('createdBy', 'name email');
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', auth, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user._id })
      .populate('assignedTo', 'name email')
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Admin: Get all tickets
router.get('/all', auth, adminAuth, async (req, res) => {
  try {
    const tickets = await Ticket.find()
      .populate('createdBy assignedTo', 'name email')
      .sort({ createdAt: -1 });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI ML Service Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
import joblib
import os
from datetime import datetime

app = FastAPI(title="Enterprise UMS ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models storage
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Load or initialize models
def load_or_create_model(name: str):
    path = f"{MODEL_PATH}/{name}.pkl"
    if os.path.exists(path):
        return joblib.load(path)
    return None

risk_model = load_or_create_model('risk_model')
burnout_model = load_or_create_model('burnout_model')

@app.get("/")
def root():
    return {"service": "Enterprise UMS ML Service", "status": "running"}
```

### Ticket Classification

```python
# ml-service/models/ticket_classifier.py
from pydantic import BaseModel
from typing import Dict
import re

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    priority: str
    confidence: float

# Simple rule-based classifier (can be replaced with ML model)
def classify_ticket(title: str, description: str) -> Dict:
    text = f"{title} {description}".lower()
    
    # Category classification
    category_keywords = {
        'technical': ['error', 'bug', 'crash', 'not working', 'broken', 'issue'],
        'account': ['login', 'password', 'access', 'permission', 'account'],
        'billing': ['payment', 'invoice', 'charge', 'subscription', 'billing'],
        'general': []
    }
    
    category_scores = {}
    for cat, keywords in category_keywords.items():
        score = sum(1 for kw in keywords if kw in text)
        category_scores[cat] = score
    
    category = max(category_scores, key=category_scores.get)
    if category_scores[category] == 0:
        category = 'general'
    
    # Priority classification
    high_priority_keywords = ['urgent', 'critical', 'emergency', 'asap', 'immediately']
    low_priority_keywords = ['minor', 'suggestion', 'question', 'inquiry']
    
    if any(kw in text for kw in high_priority_keywords):
        priority = 'high'
        confidence = 0.85
    elif any(kw in text for kw in low_priority_keywords):
        priority = 'low'
        confidence = 0.75
    else:
        priority = 'medium'
        confidence = 0.70
    
    return {
        'category': category,
        'priority': priority,
        'confidence': confidence
    }

# Add to main.py
@app.post("/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket_endpoint(request: TicketClassificationRequest):
    try:
        result = classify_ticket(request.title, request.description)
        return TicketClassificationResponse(**result)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Risk Prediction

```python
# ml-service/models/risk_predictor.py
from pydantic import BaseModel
from typing import List
import numpy as np

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_login_attempts: int
    unusual_activity_count: int
    access_violations: int
    late_tasks: int
    total_tasks: int
    last_login_days_ago: int

class RiskPredictionResponse(BaseModel):
    risk_score: float
    risk_level: str
    factors: List[str]

def predict_risk(data: RiskPredictionRequest) -> dict:
    """
    Calculate risk score based on multiple factors
    Score range: 0-100 (higher = more risk)
    """
    risk_score = 0
    factors = []
    
    # Failed login attempts
    if data.failed_login_attempts > 5:
        risk_score += 30
        factors.append('Multiple failed login attempts')
    elif data.failed_login_attempts > 2:
        risk_score += 15
        factors.append('Some failed login attempts')
    
    # Unusual activity
    if data.unusual_activity_count > 10:
        risk_score += 25
        factors.append('High unusual activity detected')
    elif data.unusual_activity_count > 5:
        risk_score += 10
        factors.append('Moderate unusual activity')
    
    # Access violations
    if data.access_violations > 0:
        risk_score += 20 * min(data.access_violations, 3)
        factors.append('Access violations detected')
    
    # Task completion rate
    if data.total_tasks > 0:
        late_rate = data.late_tasks / data.total_tasks
        if late_rate > 0.5:
            risk_score += 15
            factors.append('High rate of late tasks')
    
    # Inactive account
    if data.last_login_days_ago > 30:
        risk_score += 10
        factors.append('Account inactive for extended period')
    
    # Determine risk level
    if risk_score >= 70:
        risk_level = 'critical'
    elif risk_score >= 50:
        risk_level = 'high'
    elif risk_score >= 30:
        risk_level = 'medium'
    else:
        risk_level = 'low'
    
    return {
        'risk_score': min(risk_score, 100),
        'risk_level': risk_level,
        'factors': factors if factors else ['No significant risk factors']
    }

# Add to main.py
@app.post("/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk_endpoint(request: RiskPredictionRequest):
    try:
        result = predict_risk(request)
        return RiskPredictionResponse(**result)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Burnout Detection

```python
# ml-service/models/burnout_detector.py
from pydantic import BaseModel
from typing import List

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_daily_hours: float
    weekend_work_hours: float
    missed_deadlines: int
    stress_indicators: int  # e.g., late night activity, rapid task switching

class BurnoutDetectionResponse(BaseModel):
    burnout_score: float
    burnout_level: str
    recommendations: List[str]

def detect_burnout(data: BurnoutDetectionRequest) -> dict:
    """
    Detect employee burnout based on work patterns
    Score range: 0-100 (higher = more burnout risk)
    """
    burnout_score = 0
    recommendations = []
    
    # Workload analysis
    if data.tasks_assigned > 20:
        burnout_score += 20
        recommendations.append('Consider reducing task load')
    
    # Completion rate
    if data.tasks_assigned > 0:
        completion_rate = data.tasks_completed / data.tasks_assigned
        if completion_rate < 0.6:
            burnout_score += 15
            recommendations.append('Low task completion rate - may need support')
    
    # Working hours
    if data.avg_daily_hours > 10:
        burnout_score += 25
        recommendations.append('Excessive daily working hours detected')
    elif data.avg_daily_hours > 8:
        burnout_score += 10
    
    # Weekend work
    if data.weekend_work_hours > 5:
        burnout_score += 20
        recommendations.append('Regular weekend work - ensure proper rest')
    
    # Missed deadlines
    if data.missed_deadlines > 5:
        burnout_score += 15
        recommendations.append('Multiple missed deadlines - may indicate overload')
    
    # Stress indicators
    burnout_score += min(data.stress_indicators * 5, 25)
    if data.stress_indicators > 3:
        recommendations.append('Stress indicators present - consider wellness check-in')
    
    # Determine burnout level
    if burnout_score >= 70:
        burnout_level = 'critical'
        recommendations.insert(0, 'URGENT: Immediate intervention recommended')
    elif burnout_score >= 50:
        burnout_level = 'high'
        recommendations.insert(0, 'Schedule one-on-one meeting with manager')
    elif burnout_score >= 30:
        burnout_level = 'moderate'
    else:
        burnout_level = 'low'
        recommendations.append('Work-life balance appears healthy')
    
    return {
        'burnout_score': min(burnout_score, 100),
        'burnout_level': burnout_level,
        'recommendations': recommendations
    }

# Add to main.py
@app.post("/detect-burnout", response_model=BurnoutDetectionResponse)
async def detect_burnout_endpoint(request: BurnoutDetectionRequest):
    try:
        result = detect_burnout(request)
        return BurnoutDetectionResponse(**result)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Anomaly Detection

```python
# ml-service/models/anomaly_detector.py
from pydantic import BaseModel
from typing import List, Dict
from datetime import datetime

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    login_times: List[str]  # ISO format timestamps
    access_locations: List[str]  # IP addresses or locations
    actions_per_hour: List[int]

class AnomalyDetectionResponse(BaseModel):
    is_anomalous: bool
    anomaly_score: float
    anomalies_detected: List[str]

def detect_anomalies(data: AnomalyDetectionRequest) -> dict:
    """
    Detect anomalous behavior patterns
    """
    anomalies = []
    anomaly_score = 0
    
    # Analyze login times
    if data.login_times:
        login_hours = [datetime.fromisoformat(t.replace('Z', '+00:00')).hour 
                      for t in data.login_times]
        
        # Check for unusual login times (late night/early morning)
        unusual_hours = [h for h in login_hours if h < 6 or h > 22]
        if len(unusual_hours) > len(login_hours) * 0.3:
            anomalies.append('Unusual login times detected')
            anomaly_score += 30
    
    # Analyze access locations
    if len(data.access_locations) > len(set(data.access_locations)):
        unique_locations = len(set(data.access_locations))
        if unique_locations > 5:
            anomalies.append('Multiple access locations')
            anomaly_score += 25
    
    # Analyze activity patterns
    if data.actions_per_hour:
        avg_actions = sum(data.actions_per_hour) / len(data.actions_per_hour)
        max_actions = max(data.actions_per_hour)
        
        # Detect spikes in activity
        if max_actions > avg_actions * 3:
            anomalies.append('Unusual spike in activity detected')
            anomaly_score += 20
        
        # Detect sustained high activity
        if avg_actions > 100:
            anomalies.append('Sustained high activity rate')
            anomaly_score += 15
    
    is_anomalous = anomaly_score >= 30
    
    return {
        'is_anomalous': is_anomalous,
        'anomaly_score': min(anomaly_score, 100),
        'anomalies_detected': anomalies if anomalies else ['No anomalies detected']
    }

# Add to main.py
@app.post("/detect-anomalies", response_model=AnomalyDetectionResponse)
async def detect_anomalies_endpoint(request: AnomalyDetectionRequest):
    try:
        result = detect_anomalies(request)
        return AnomalyDetectionResponse(**result)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend Integration

### API Service Setup

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

// Create axios instance
const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Add auth token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auth APIs
export const authAPI = {
  login: (credentials) => api.post('/api/auth/login', credentials),
  register: (userData) => api.post('/api/auth/register', userData),
  getCurrentUser: () => api.get('/api/auth/me')
};

// Task APIs
export const taskAPI = {
  getMyTasks: () => api.get('/api/tasks/my-tasks'),
  createTask: (taskData) => api.post('/api/tasks', taskData),
  updateTaskStatus: (taskId, status) => 
    api.patch(`/api/tasks/${taskId}/status`, { status }),
  trackTime: (taskId, minutes) => 
    api.patch(`/api/tasks/${taskId}/time`, { minutes })
};

// Ticket APIs
export const ticketAPI = {
  createTicket: (ticketData) => api.post('/api/tickets', ticketData),
  getMyTickets: () => api.get('/api/tickets/my-tickets'),
  getAllTickets: () => api.get('/api/tickets/all')
};

// ML APIs
export const mlAPI = {
  classifyTicket: (title, description) => 
    axios.post(`${ML_URL}/classify-ticket`, { title, description }),
  
  predictRisk: (userData) => 
    axios.post(`${ML_URL}/predict-risk`, userData),
  
  detectBurnout: (workData) => 
    axios.post(`${ML_URL}/detect-burnout`, workData),
  
  detectAnomalies: (behaviorData) => 
    axios.post(`${ML_URL}/detect-anomalies`, behaviorData)
};

export default api;
```

### React Components

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';
import './TaskBoard.css';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    try {
      const response = await taskAPI.getMyTasks();
