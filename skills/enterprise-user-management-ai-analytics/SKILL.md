---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management using React, Node.js, and FastAPI ML service
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics for user risk detection
  - build a user management dashboard with task tracking
  - implement JWT authentication for enterprise apps
  - create AI-powered ticket classification system
  - set up burnout detection and anomaly detection
  - configure MongoDB for user and task management
  - deploy enterprise user management with ML service
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines traditional user/task management with AI-powered insights. The system provides role-based access control (admin/user), task tracking with Kanban boards, support ticket management, and ML-driven features including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

**Architecture:**
- **Frontend:** React.js with JWT authentication
- **Backend:** Node.js REST API
- **ML Service:** FastAPI with scikit-learn and River for online learning
- **Database:** MongoDB

## Installation

### Prerequisites

```bash
# Required
node --version  # v14+
python --version  # 3.8+
mongod --version  # MongoDB 4.4+
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# ML Service setup
cd ../ml-service
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Frontend setup
cd ../frontend
npm install
# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF
```

## Running the System

### Start All Services

```bash
# Terminal 1: MongoDB
mongod --dbpath /path/to/data/db

# Terminal 2: Backend
cd backend
npm start
# Runs on http://localhost:5000

# Terminal 3: ML Service
cd ml-service
source venv/bin/activate
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# Runs on http://localhost:8000

# Terminal 4: Frontend
cd frontend
npm start
# Runs on http://localhost:3000
```

### Development Mode

```bash
# Backend with hot reload
cd backend
npm run dev

# Frontend with hot reload (already default)
cd frontend
npm start
```

## Backend API Usage

### Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// User registration
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const user = new User({
      username,
      email,
      password, // Should be hashed in User model pre-save hook
      role: role || 'user'
    });

    await user.save();

    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );

    res.status(201).json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// User login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    const isValidPassword = await user.comparePassword(password);
    if (!isValidPassword) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }

    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );

    res.json({
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});
```

### Middleware for Auth Protection

```javascript
// backend/middleware/auth.js
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

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Task Management API

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

// Create task (Admin only)
router.post('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, deadline } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      priority,
      deadline,
      status: 'To Do',
      createdBy: req.user.userId
    });

    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user tasks
router.get('/my-tasks', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const validStatuses = ['To Do', 'In Progress', 'Done'];
    
    if (!validStatuses.includes(status)) {
      return res.status(400).json({ message: 'Invalid status' });
    }

    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    if (task.assignedTo.toString() !== req.user.userId && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Access denied' });
    }

    task.status = status;
    if (status === 'Done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Track time on task
router.post('/:id/time-log', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body; // duration in seconds
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    task.timeSpent = (task.timeSpent || 0) + duration;
    await task.save();
    
    res.json({ message: 'Time logged', totalTime: task.timeSpent });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Support Ticket API

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const Ticket = require('../models/Ticket');
const axios = require('axios');
const { authMiddleware } = require('../middleware/auth');

// Create support ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Call ML service for ticket classification
    let category = 'general';
    let suggestedPriority = priority;
    
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/classify-ticket`,
        { title, description }
      );
      category = mlResponse.data.category;
      suggestedPriority = mlResponse.data.priority;
    } catch (mlError) {
      console.error('ML classification failed:', mlError.message);
    }

    const ticket = new Ticket({
      title,
      description,
      category,
      priority: suggestedPriority,
      status: 'open',
      createdBy: req.user.userId
    });

    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.userId })
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: error.message });
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
from sklearn.feature_extraction.text import TfidfVectorizer
import joblib
import os
from datetime import datetime

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models (loaded or initialized)
ticket_classifier = None
vectorizer = None
anomaly_detector = None

@app.on_event("startup")
async def load_models():
    global ticket_classifier, vectorizer, anomaly_detector
    
    model_path = os.getenv('MODEL_PATH', './models')
    os.makedirs(model_path, exist_ok=True)
    
    # Initialize or load ticket classifier
    try:
        ticket_classifier = joblib.load(f'{model_path}/ticket_classifier.pkl')
        vectorizer = joblib.load(f'{model_path}/vectorizer.pkl')
    except:
        vectorizer = TfidfVectorizer(max_features=100)
        ticket_classifier = RandomForestClassifier(n_estimators=100)
        print("Initialized new ticket classifier")
    
    # Initialize anomaly detector
    anomaly_detector = IsolationForest(contamination=0.1, random_state=42)
    print("ML models loaded successfully")

class TicketInput(BaseModel):
    title: str
    description: str

class TicketClassification(BaseModel):
    category: str
    priority: str
    confidence: float

@app.post("/classify-ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketInput):
    """Classify support ticket using NLP"""
    try:
        text = f"{ticket.title} {ticket.description}".lower()
        
        # Simple rule-based classification (replace with trained model)
        categories = {
            'technical': ['bug', 'error', 'crash', 'issue', 'problem', 'not working'],
            'account': ['login', 'password', 'access', 'permission', 'account'],
            'feature': ['request', 'feature', 'add', 'new', 'enhancement'],
            'general': []
        }
        
        category = 'general'
        for cat, keywords in categories.items():
            if any(keyword in text for keyword in keywords):
                category = cat
                break
        
        # Determine priority
        high_priority_keywords = ['urgent', 'critical', 'emergency', 'asap', 'immediately']
        priority = 'high' if any(kw in text for kw in high_priority_keywords) else 'medium'
        
        return TicketClassification(
            category=category,
            priority=priority,
            confidence=0.85
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class UserBehavior(BaseModel):
    user_id: str
    login_count: int
    failed_logins: int
    unusual_hours: int
    location_changes: int
    data_access_volume: float

class RiskPrediction(BaseModel):
    risk_score: float
    risk_level: str
    factors: List[str]

@app.post("/predict-risk", response_model=RiskPrediction)
async def predict_user_risk(behavior: UserBehavior):
    """Predict user security risk based on behavior"""
    try:
        # Calculate risk score (0-100)
        risk_score = 0.0
        factors = []
        
        # Failed login attempts
        if behavior.failed_logins > 3:
            risk_score += 25
            factors.append("Multiple failed login attempts")
        
        # Unusual hours access
        if behavior.unusual_hours > 5:
            risk_score += 20
            factors.append("Frequent unusual hours access")
        
        # Location changes
        if behavior.location_changes > 3:
            risk_score += 15
            factors.append("Multiple location changes")
        
        # High data access
        if behavior.data_access_volume > 1000:
            risk_score += 30
            factors.append("Unusually high data access volume")
        
        # Login frequency
        if behavior.login_count > 50:
            risk_score += 10
            factors.append("Very high login frequency")
        
        # Determine risk level
        if risk_score >= 70:
            risk_level = "high"
        elif risk_score >= 40:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        return RiskPrediction(
            risk_score=min(risk_score, 100.0),
            risk_level=risk_level,
            factors=factors if factors else ["No significant risk factors detected"]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class WorkloadData(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    hours_worked: float
    overtime_hours: float
    missed_deadlines: int

class BurnoutAnalysis(BaseModel):
    burnout_score: float
    burnout_risk: str
    recommendations: List[str]

@app.post("/detect-burnout", response_model=BurnoutAnalysis)
async def detect_burnout(workload: WorkloadData):
    """Detect employee burnout risk"""
    try:
        burnout_score = 0.0
        recommendations = []
        
        # Task completion rate
        completion_rate = workload.tasks_completed / max(workload.tasks_assigned, 1)
        if completion_rate < 0.6:
            burnout_score += 25
            recommendations.append("Low task completion rate - consider redistributing workload")
        
        # Overtime analysis
        if workload.overtime_hours > 10:
            burnout_score += 30
            recommendations.append("Excessive overtime detected - schedule time off")
        
        # Missed deadlines
        if workload.missed_deadlines > 3:
            burnout_score += 20
            recommendations.append("Frequent missed deadlines - provide additional support")
        
        # Work hours
        if workload.hours_worked > 50:
            burnout_score += 25
            recommendations.append("Long work hours - encourage work-life balance")
        
        # Determine risk level
        if burnout_score >= 60:
            burnout_risk = "high"
        elif burnout_score >= 30:
            burnout_risk = "medium"
        else:
            burnout_risk = "low"
            recommendations.append("Workload appears manageable")
        
        return BurnoutAnalysis(
            burnout_score=min(burnout_score, 100.0),
            burnout_risk=burnout_risk,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class AnomalyDetectionInput(BaseModel):
    features: List[float]

class AnomalyResult(BaseModel):
    is_anomaly: bool
    anomaly_score: float

@app.post("/detect-anomaly", response_model=AnomalyResult)
async def detect_anomaly(data: AnomalyDetectionInput):
    """Detect anomalies in user behavior"""
    try:
        global anomaly_detector
        
        features = np.array(data.features).reshape(1, -1)
        
        # Predict (-1 for anomaly, 1 for normal)
        prediction = anomaly_detector.predict(features)[0]
        score = anomaly_detector.score_samples(features)[0]
        
        return AnomalyResult(
            is_anomaly=prediction == -1,
            anomaly_score=float(abs(score))
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "timestamp": datetime.now().isoformat()}
```

## Frontend Integration

### API Service Layer

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

// Create axios instance with auth interceptor
const apiClient = axios.create({
  baseURL: API_URL,
});

apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Auth API
export const authAPI = {
  login: (credentials) => apiClient.post('/auth/login', credentials),
  register: (userData) => apiClient.post('/auth/register', userData),
  getCurrentUser: () => apiClient.get('/auth/me'),
};

// Task API
export const taskAPI = {
  getMyTasks: () => apiClient.get('/tasks/my-tasks'),
  getAllTasks: () => apiClient.get('/tasks'),
  createTask: (taskData) => apiClient.post('/tasks', taskData),
  updateTaskStatus: (taskId, status) => 
    apiClient.patch(`/tasks/${taskId}/status`, { status }),
  logTime: (taskId, duration) => 
    apiClient.post(`/tasks/${taskId}/time-log`, { duration }),
};

// Ticket API
export const ticketAPI = {
  getMyTickets: () => apiClient.get('/tickets/my-tickets'),
  getAllTickets: () => apiClient.get('/tickets'),
  createTicket: (ticketData) => apiClient.post('/tickets', ticketData),
  updateTicket: (ticketId, updates) => 
    apiClient.patch(`/tickets/${ticketId}`, updates),
};

// ML Service API
export const mlAPI = {
  classifyTicket: (title, description) =>
    axios.post(`${ML_API_URL}/classify-ticket`, { title, description }),
  predictRisk: (behaviorData) =>
    axios.post(`${ML_API_URL}/predict-risk`, behaviorData),
  detectBurnout: (workloadData) =>
    axios.post(`${ML_API_URL}/detect-burnout`, workloadData),
  detectAnomaly: (features) =>
    axios.post(`${ML_API_URL}/detect-anomaly`, { features }),
};

export default apiClient;
```

### React Task Dashboard Component

```javascript
// frontend/src/components/TaskDashboard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';
import './TaskDashboard.css';

const TaskDashboard = () => {
  const [tasks, setTasks] = useState({ toDo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);
  const [activeTimer, setActiveTimer] = useState(null);
  const [timerSeconds, setTimerSeconds] = useState(0);

  useEffect(() => {
    fetchTasks();
  }, []);

  useEffect(() => {
    let interval = null;
    if (activeTimer) {
      interval = setInterval(() => {
        setTimerSeconds((seconds) => seconds + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [activeTimer]);

  const fetchTasks = async () => {
    try {
      const response = await taskAPI.getMyTasks();
      const tasksByStatus = {
        toDo: response.data.filter(t => t.status === 'To Do'),
        inProgress: response.data.filter(t => t.status === 'In Progress'),
        done: response.data.filter(t => t.status === 'Done'),
      };
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await taskAPI.updateTaskStatus(taskId, newStatus);
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const startTimer = (taskId) => {
    if (activeTimer && activeTimer !== taskId) {
      stopTimer(activeTimer);
    }
    setActiveTimer(taskId);
    setTimerSeconds(0);
  };

  const stopTimer = async (taskId) => {
    if (timerSeconds > 0) {
      try {
        await taskAPI.logTime(taskId, timerSeconds);
      } catch (error) {
        console.error('Failed to log time:', error);
      }
    }
    setActiveTimer(null);
    setTimerSeconds(0);
  };

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const TaskCard = ({ task }) => (
    <div className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority priority-${task.priority}`}>
          {task.priority}
        </span>
        {task.deadline && (
          <span className="deadline">
            Due: {new Date(task.deadline).toLocaleDateString()}
          </span>
        )}
      </div>
      
      <div className="task-actions">
        {task.status !== 'Done' && (
          <>
            {activeTimer === task._id ? (
              <>
                <span className="timer">{formatTime(timerSeconds)}</span>
                <button onClick={() => stopTimer(task._id)}>Stop</button>
              </>
            ) : (
              <button onClick={() => startTimer(task._id)}>Start Timer</button>
            )}
          </>
        )}
        
        <select
          value={task.status}
          onChange={(e) => handleStatusChange(task._id, e.target.value)}
        >
          <option value="To Do">To Do</option>
          <option value="In Progress">In Progress</option>
          <option value="Done">Done</option>
        </select>
      </div>
    </div>
  );

  if (loading) return <div className="loading">Loading tasks...</div>;

  return (
    <div className="task-dashboard">
      <h2>My Tasks</h2>
      <div className="kanban-board">
        <div className="kanban-column">
          <h3>To Do ({tasks.toDo.length})</h3>
          {tasks.toDo.map(task => <TaskCard key={task._id} task={task} />)}
        </div>
        
        <div className="kanban-column">
          <h3>In Progress ({tasks.inProgress.length})</h3>
          {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
        </div>
        
        <div className="kanban-column">
          <h3>Done ({tasks.done.length})</h3>
          {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
        </div>
      </div>
    </div>
  );
};

export default TaskDashboard;
```

### AI-Powered Ticket Creation

```javascript
// frontend/src/components/CreateTicket.jsx
import React, { useState } from 'react';
import { ticketAPI, mlAPI } from '../services/api';

const CreateTicket = ({ onTicketCreated }) => {
  const [formData, setFormData] = useState({
    title: '',
    description: '',
    priority: 'medium',
  });
  const [aiSuggestion, setAiSuggestion] = useState(null);
  const [loading, setLoading] = useState(false);

  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const getAISuggestions = async () => {
    if (!formData.title || !formData.description) return;
    
    try {
      const response = await mlAPI.classifyTicket(
        formData.title,
        formData.description
      );
      setAiSuggestion(response.data);
    } catch (error) {
      console.error('AI classification failed:', error);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);

    try {
      const ticketData = {
        ...formData,
        priority: aiSuggestion?.priority || formData.priority,
      };
      
      await ticketAPI.createTicket(ticketData);
      
      // Reset form
      setFormData({ title: '', description: '', priority: 'medium' });
      setAiSuggestion(null);
      
      if (onTicketCreated) onTicketCreated();
    } catch (error) {
      console.error('Failed to create ticket:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="create-ticket">
      <h3>Create Support Ticket</h3>
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label>Title</label>
          <input
            type="text"
            name="title"
            value={formData.title}
            onChange={handleChange}
            required
          />
        </div>

        <div className="form-group">
          <label>Description</label>
          <textarea
            name="description"
            value={formData.description}
            onChange={handleChange}
            rows="5"
            required
          />
        </div>

        <button
          type="button"
          onClick={getAISuggestions}
          className="btn-secondary"
        >
          Get AI Suggestions
        </button>

        {aiSuggestion && (
          <div className="ai-suggestion">
            <h4>AI Classification</h4>
            <p><strong>Category:</strong> {aiSuggestion.category}</p>
            <p><strong>Suggested Priority:</strong> {aiSuggestion.priority}</p>
            <p><strong>Confidence:</strong> {(aiSuggestion.confidence * 100).toFixed(1)}%</p>
          </div>
        )}

        <div className="form-group">
          <label>Priority</label>
          <select
            name="priority"
            value={formData.priority}
            onChange={handleChange}
          >
            <option value="low">Low</option>
            <option value="medium">Medium</option>
            <option value="high">High</option>
          </select>
        </div>

        <button type="submit" disabled={loading} className="btn-primary">
          {loading ? 'Creating...' : 'Create Ticket'}
        </button>
      </form>
    </div>
  );
};

export default CreateTicket;
```

## MongoDB Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
