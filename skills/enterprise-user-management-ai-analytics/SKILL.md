---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly analysis, and intelligent ticket routing
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics into user management
  - implement intelligent ticket classification
  - build user management with risk detection
  - create admin dashboard with AI insights
  - set up anomaly detection for user behavior
  - deploy user management system with ML service
  - configure JWT authentication for enterprise app
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack JavaScript application that combines traditional user/task management with AI-powered analytics. It provides role-based access control, Kanban-style task tracking, support ticket management, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

**Key capabilities:**
- User authentication with JWT
- Admin and user role management
- Task assignment and Kanban boards
- Support ticket system with AI classification
- ML-based risk and anomaly detection
- Real-time analytics and alerts

## Installation

### Prerequisites

```bash
# Node.js 14+ and Python 3.8+ required
node --version
python --version
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

### ML Service Setup (FastAPI)

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=info
EOF

uvicorn main:app --reload --port 8000
```

### Frontend Setup (React)

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

npm start
```

## Architecture

The system consists of three main services:

1. **Frontend (React)** - Port 3000
2. **Backend (Node.js/Express)** - Port 5000
3. **ML Service (FastAPI)** - Port 8000

## Backend API Usage

### Authentication

```javascript
// backend/controllers/authController.js
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// Register user
const register = async (req, res) => {
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
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.status(201).json({ token, user: { id: user._id, username, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Login user
const login = async (req, res) => {
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
      { expiresIn: '24h' }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

module.exports = { register, login };
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authenticate = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authenticate, requireAdmin };
```

### User Management

```javascript
// backend/controllers/userController.js
const User = require('../models/User');

// Get all users (admin only)
const getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Update user
const updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;
    
    // Prevent password update through this endpoint
    delete updates.password;
    
    const user = await User.findByIdAndUpdate(
      id,
      updates,
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Delete user
const deleteUser = async (req, res) => {
  try {
    const { id } = req.params;
    
    const user = await User.findByIdAndDelete(id);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

module.exports = { getAllUsers, updateUser, deleteUser };
```

### Task Management

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
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
  dueDate: { type: Date },
  timeSpent: { type: Number, default: 0 }, // in minutes
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

// Create task
const createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    
    await task.save();
    await task.populate('assignedTo', 'username email');
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Get user tasks
const getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'username')
      .sort('-createdAt');
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Update task status
const updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status, timeSpent } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      id,
      { 
        status, 
        timeSpent,
        updatedAt: Date.now() 
      },
      { new: true }
    ).populate('assignedTo', 'username email');
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

module.exports = { createTask, getUserTasks, updateTaskStatus };
```

### Support Ticket System

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { 
    type: String, 
    enum: ['technical', 'billing', 'general', 'urgent'],
    default: 'general'
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
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: { type: Object },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

// Create ticket with AI classification
const createTicket = async (req, res) => {
  try {
    const { title, description } = req.body;
    
    // Get AI classification
    let aiClassification = null;
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        text: `${title} ${description}`
      });
      aiClassification = mlResponse.data;
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      createdBy: req.user.id,
      category: aiClassification?.category || 'general',
      priority: aiClassification?.priority || 'medium',
      aiClassification
    });
    
    await ticket.save();
    await ticket.populate('createdBy', 'username email');
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Get tickets
const getTickets = async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.id };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort('-createdAt');
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

module.exports = { createTicket, getTickets };
```

## ML Service API

### FastAPI Application Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import pickle
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS middleware
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

# Global model instances
ticket_classifier = None
risk_detector = None
anomaly_detector = None

class TicketClassificationRequest(BaseModel):
    text: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_logins: int
    unusual_access_time: bool
    data_access_volume: float
    permission_changes: int

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    features: List[float]

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_completed: int
    avg_task_time: float
    overtime_hours: float
    days_without_break: int

@app.on_event("startup")
async def load_models():
    """Load or initialize ML models on startup"""
    global ticket_classifier, risk_detector, anomaly_detector
    
    # Initialize ticket classifier (simple example)
    ticket_classifier = {
        'vectorizer': TfidfVectorizer(max_features=100),
        'model': MultinomialNB()
    }
    
    # Load pre-trained models if they exist
    try:
        with open(f'{MODEL_PATH}/ticket_classifier.pkl', 'rb') as f:
            ticket_classifier = pickle.load(f)
    except FileNotFoundError:
        print("Ticket classifier not found, using default")

@app.get("/")
async def root():
    return {"message": "Enterprise User Management ML Service", "status": "running"}

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket into category and priority"""
    try:
        text = request.text.lower()
        
        # Simple keyword-based classification
        category = 'general'
        priority = 'medium'
        
        # Category detection
        if any(word in text for word in ['bug', 'error', 'crash', 'broken', 'not working']):
            category = 'technical'
            priority = 'high'
        elif any(word in text for word in ['payment', 'invoice', 'billing', 'charge']):
            category = 'billing'
        elif any(word in text for word in ['urgent', 'critical', 'emergency', 'asap']):
            category = 'urgent'
            priority = 'critical'
        
        return {
            'category': category,
            'priority': priority,
            'confidence': 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior"""
    try:
        # Simple risk scoring algorithm
        risk_score = 0.0
        
        # Failed login attempts
        risk_score += min(request.failed_logins * 15, 40)
        
        # Unusual access time
        if request.unusual_access_time:
            risk_score += 20
        
        # High data access volume
        if request.data_access_volume > 1000:
            risk_score += 25
        
        # Permission changes
        risk_score += min(request.permission_changes * 10, 30)
        
        risk_score = min(risk_score, 100)
        
        risk_level = 'low'
        if risk_score > 70:
            risk_level = 'critical'
        elif risk_score > 50:
            risk_level = 'high'
        elif risk_score > 30:
            risk_level = 'medium'
        
        return {
            'user_id': request.user_id,
            'risk_score': risk_score,
            'risk_level': risk_level,
            'recommendations': get_risk_recommendations(risk_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_risk_recommendations(risk_level: str) -> List[str]:
    """Get recommendations based on risk level"""
    recommendations = {
        'low': ['Continue monitoring user activity'],
        'medium': ['Enable additional logging', 'Review recent activities'],
        'high': ['Require password reset', 'Enable MFA', 'Alert security team'],
        'critical': ['Suspend account', 'Immediate security review', 'Notify admin']
    }
    return recommendations.get(risk_level, [])

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    try:
        features = np.array(request.features)
        
        # Simple threshold-based anomaly detection
        mean = np.mean(features)
        std = np.std(features)
        
        is_anomaly = False
        for value in features:
            if abs(value - mean) > 2 * std:
                is_anomaly = True
                break
        
        return {
            'user_id': request.user_id,
            'is_anomaly': is_anomaly,
            'anomaly_score': float(abs(max(features) - mean) / (std + 1e-6)),
            'alert': is_anomaly
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze employee burnout risk"""
    try:
        burnout_score = 0.0
        
        # Task completion rate (more than 15 tasks/day is high)
        if request.tasks_completed > 15:
            burnout_score += 25
        
        # Average task time (longer than 4 hours is concerning)
        if request.avg_task_time > 4.0:
            burnout_score += 20
        
        # Overtime hours
        burnout_score += min(request.overtime_hours * 2, 30)
        
        # Days without break
        burnout_score += min(request.days_without_break * 5, 25)
        
        burnout_score = min(burnout_score, 100)
        
        risk_level = 'low'
        if burnout_score > 70:
            risk_level = 'high'
        elif burnout_score > 40:
            risk_level = 'medium'
        
        return {
            'user_id': request.user_id,
            'burnout_score': burnout_score,
            'risk_level': risk_level,
            'recommendations': get_burnout_recommendations(risk_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def get_burnout_recommendations(risk_level: str) -> List[str]:
    """Get burnout prevention recommendations"""
    recommendations = {
        'low': ['Maintain healthy work-life balance'],
        'medium': ['Consider taking a break', 'Reduce overtime hours', 'Delegate tasks'],
        'high': ['Mandatory time off recommended', 'Workload redistribution needed', 'Schedule wellness check']
    }
    return recommendations.get(risk_level, [])

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Frontend Implementation

### React API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

// Create axios instance
const api = axios.create({
  baseURL: API_URL,
});

// Add auth token to requests
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Auth API
export const authAPI = {
  login: (credentials) => api.post('/auth/login', credentials),
  register: (userData) => api.post('/auth/register', userData),
  logout: () => {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  }
};

// User API
export const userAPI = {
  getAllUsers: () => api.get('/users'),
  updateUser: (id, data) => api.put(`/users/${id}`, data),
  deleteUser: (id) => api.delete(`/users/${id}`),
  getCurrentUser: () => api.get('/users/me')
};

// Task API
export const taskAPI = {
  getTasks: () => api.get('/tasks'),
  createTask: (taskData) => api.post('/tasks', taskData),
  updateTask: (id, data) => api.put(`/tasks/${id}`, data),
  deleteTask: (id) => api.delete(`/tasks/${id}`)
};

// Ticket API
export const ticketAPI = {
  getTickets: () => api.get('/tickets'),
  createTicket: (ticketData) => api.post('/tickets', ticketData),
  updateTicket: (id, data) => api.put(`/tickets/${id}`, data)
};

// ML API
export const mlAPI = {
  classifyTicket: (text) => axios.post(`${ML_API_URL}/classify-ticket`, { text }),
  predictRisk: (userData) => axios.post(`${ML_API_URL}/predict-risk`, userData),
  detectAnomaly: (data) => axios.post(`${ML_API_URL}/detect-anomaly`, data),
  analyzeBurnout: (data) => axios.post(`${ML_API_URL}/analyze-burnout`, data)
};

export default api;
```

### User Dashboard Component

```javascript
// frontend/src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI, mlAPI } from '../services/api';
import './UserDashboard.css';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [burnoutAnalysis, setBurnoutAnalysis] = useState(null);
  const [activeTimer, setActiveTimer] = useState(null);
  const [timeSpent, setTimeSpent] = useState(0);

  useEffect(() => {
    loadTasks();
    loadBurnoutAnalysis();
  }, []);

  const loadTasks = async () => {
    try {
      const response = await taskAPI.getTasks();
      setTasks(response.data);
    } catch (error) {
      console.error('Error loading tasks:', error);
    }
  };

  const loadBurnoutAnalysis = async () => {
    try {
      const user = JSON.parse(localStorage.getItem('user'));
      const response = await mlAPI.analyzeBurnout({
        user_id: user.id,
        tasks_completed: 12,
        avg_task_time: 3.5,
        overtime_hours: 10,
        days_without_break: 5
      });
      setBurnoutAnalysis(response.data);
    } catch (error) {
      console.error('Error loading burnout analysis:', error);
    }
  };

  const startTimer = (taskId) => {
    setActiveTimer(taskId);
    const interval = setInterval(() => {
      setTimeSpent(prev => prev + 1);
    }, 60000); // Increment every minute
    
    return () => clearInterval(interval);
  };

  const stopTimer = async (taskId) => {
    setActiveTimer(null);
    try {
      await taskAPI.updateTask(taskId, {
        status: 'in-progress',
        timeSpent: timeSpent
      });
      setTimeSpent(0);
      loadTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await taskAPI.updateTask(taskId, { status: newStatus });
      loadTasks();
    } catch (error) {
      console.error('Error moving task:', error);
    }
  };

  const getTasksByStatus = (status) => {
    return tasks.filter(task => task.status === status);
  };

  return (
    <div className="user-dashboard">
      <h1>My Dashboard</h1>
      
      {burnoutAnalysis && (
        <div className={`burnout-alert ${burnoutAnalysis.risk_level}`}>
          <h3>Burnout Risk: {burnoutAnalysis.risk_level.toUpperCase()}</h3>
          <p>Score: {burnoutAnalysis.burnout_score}/100</p>
          <ul>
            {burnoutAnalysis.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      )}

      <div className="kanban-board">
        <div className="kanban-column">
          <h3>To Do</h3>
          {getTasksByStatus('todo').map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => moveTask(task._id, 'in-progress')}>
                Start
              </button>
            </div>
          ))}
        </div>

        <div className="kanban-column">
          <h3>In Progress</h3>
          {getTasksByStatus('in-progress').map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <p>Time: {task.timeSpent} minutes</p>
              {activeTimer === task._id ? (
                <button onClick={() => stopTimer(task._id)}>Stop Timer</button>
              ) : (
                <button onClick={() => startTimer(task._id)}>Start Timer</button>
              )}
              <button onClick={() => moveTask(task._id, 'done')}>Complete</button>
            </div>
          ))}
        </div>

        <div className="kanban-column">
          <h3>Done</h3>
          {getTasksByStatus('done').map(task => (
            <div key={task._id} className="task-card completed">
              <h4>{task.title}</h4>
              <p>Completed in {task.timeSpent} minutes</p>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Dashboard with Analytics

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import { userAPI, taskAPI, ticketAPI, mlAPI } from '../services/api';
import './AdminDashboard.css';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [tasks, setTasks] = useState([]);
  const [tickets, setTickets] = useState([]);
  const [riskAlerts, setRiskAlerts] = useState([]);

  useEffect(() => {
    loadAllData();
  }, []);

  const loadAllData = async () => {
    try {
      const [usersRes, tasksRes, ticketsRes] = await Promise.all([
        userAPI.getAllUsers(),
        taskAPI.getTasks(),
        ticketAPI.getTickets()
      ]);
      
      setUsers(usersRes.data);
      setTasks(tasksRes.data);
      setTickets(ticketsRes.data);
      
      // Run risk analysis for each user
      analyzeUserRisks(usersRes.data);
    } catch (error) {
      console.error('Error loading data:', error);
    }
  };

  const analyzeUserRisks = async (usersList) => {
    const alerts = [];
    
    for (const user of usersList) {
      try {
        const riskData = await mlAPI.predictRisk({
          user_id: user._id,
          failed_logins: user.failedLoginAttempts || 0,
          unusual_access_time: user.lastLoginTime ? isUnusualTime(user.lastLoginTime) : false,
          data_access_volume: user.dataAccessVolume || 0,
          permission_changes: user.permissionChanges || 0
        });
        
        if (riskData.data.risk_level === 'high' || riskData.data.risk_level === 'critical') {
          alerts.push({
            user: user.username,
            ...riskData.data
          });
        }
      } catch (error) {
        console.error(`Error analyzing risk for user ${user._id}:`, error);
      }
    }
    
    setRiskAlerts(alerts);
  };

  const isUnusualTime = (loginTime) => {
    const hour = new Date(loginTime).getHours();
    return hour < 6 || hour > 22;
  };

  const handleDeleteUser = async (userId) => {
    if (window.confirm('Are you sure you want to delete this user?')) {
      try {
        await userAPI.deleteUser(userId);
        loadAllData();
      
