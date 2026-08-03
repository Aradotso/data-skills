---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket classification, and risk detection
triggers:
  - "setup enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "add AI ticket classification system"
  - "build admin panel for user management"
  - "integrate burnout detection and risk prediction"
  - "setup kanban board with time tracking"
  - "configure ML service for anomaly detection"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines user/task management with AI-powered analytics. It provides role-based access control, Kanban task boards, support ticket management, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and ticket auto-classification.

**Core Components:**
- **Frontend**: React.js dashboard for users and admins
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based AI analytics using scikit-learn and River
- **Database**: MongoDB for user, task, and ticket storage

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

# Backend setup
cd backend
npm install
cp .env.example .env  # Configure environment variables

# ML Service setup
cd ../ml-service
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt

# Frontend setup
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env):**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env):**
```bash
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

**Frontend (.env):**
```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

## Running the Application

```bash
# Terminal 1: Backend
cd backend
npm start

# Terminal 2: ML Service
cd ml-service
uvicorn main:app --reload --port 8000

# Terminal 3: Frontend
cd frontend
npm start
```

Access points:
- Frontend: http://localhost:3000
- Backend API: http://localhost:5000
- ML Service: http://localhost:8000/docs (Swagger UI)

## Backend API Structure

### Authentication Endpoints

```javascript
// Register new user
POST /api/auth/register
Content-Type: application/json

{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"  // "user" or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "507f1f77bcf86cd799439011",
    "username": "john_doe",
    "role": "user"
  }
}
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Authorization: Bearer ${JWT_TOKEN}

// Create user
POST /api/users
Authorization: Bearer ${JWT_TOKEN}
{
  "username": "jane_smith",
  "email": "jane@example.com",
  "password": "password123",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Authorization: Bearer ${JWT_TOKEN}
{
  "role": "admin",
  "department": "Management"
}

// Delete user
DELETE /api/users/:userId
Authorization: Bearer ${JWT_TOKEN}
```

### Task Management

```javascript
// Create task
POST /api/tasks
Authorization: Bearer ${JWT_TOKEN}
{
  "title": "Implement feature X",
  "description": "Add new analytics dashboard",
  "assignedTo": "507f1f77bcf86cd799439011",
  "priority": "high",  // "low", "medium", "high"
  "dueDate": "2026-05-01T00:00:00Z",
  "status": "todo"  // "todo", "inprogress", "done"
}

// Update task status
PATCH /api/tasks/:taskId/status
{
  "status": "inprogress"
}

// Track time on task
POST /api/tasks/:taskId/time
{
  "duration": 3600  // seconds
}

// Get user tasks
GET /api/tasks/user/:userId
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Authorization: Bearer ${JWT_TOKEN}
{
  "subject": "Login issue",
  "description": "Cannot access dashboard after password reset",
  "priority": "high",
  "category": "technical"
}

// AI auto-classification will categorize and route ticket

// Update ticket
PATCH /api/tickets/:ticketId
{
  "status": "resolved",
  "resolution": "Password reset link sent"
}

// Get tickets
GET /api/tickets?status=open&priority=high
```

## ML Service API

### AI Ticket Classification

```python
# FastAPI endpoint
@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketInput):
    """
    Classifies ticket category and priority using ML
    """
    category = ticket_classifier.predict(ticket.description)
    priority = priority_model.predict(ticket.subject)
    
    return {
        "category": category,
        "priority": priority,
        "confidence": 0.87
    }
```

```javascript
// Frontend/Backend usage
const classifyTicket = async (ticketData) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description
    })
  });
  
  const result = await response.json();
  // { category: "technical", priority: "high", confidence: 0.87 }
  return result;
};
```

### Risk Detection

```python
@app.post("/api/ml/risk-detection")
async def detect_risk(user_behavior: UserBehavior):
    """
    Detects anomalous user behavior patterns
    """
    features = extract_features(user_behavior)
    risk_score = risk_model.predict_proba(features)
    
    return {
        "risk_level": "medium" if risk_score > 0.6 else "low",
        "risk_score": risk_score,
        "factors": ["unusual_login_time", "high_failed_attempts"]
    }
```

```javascript
// Check user risk
const checkUserRisk = async (userId) => {
  const behavior = await getUserBehavior(userId);
  
  const response = await fetch('http://localhost:8000/api/ml/risk-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      loginPatterns: behavior.logins,
      activityLog: behavior.activities,
      failedAttempts: behavior.failedLogins
    })
  });
  
  return await response.json();
};
```

### Burnout Detection

```python
@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(workload_data: WorkloadData):
    """
    Analyzes user workload to detect burnout risk
    """
    features = {
        'avg_daily_hours': workload_data.avg_hours,
        'task_count': workload_data.task_count,
        'overdue_tasks': workload_data.overdue_count,
        'stress_indicators': workload_data.stress_score
    }
    
    burnout_risk = burnout_model.predict(features)
    
    return {
        "burnout_risk": burnout_risk,
        "recommendation": "Reduce workload by 20%",
        "score": 0.73
    }
```

```javascript
// Monitor team burnout
const analyzeBurnout = async (userId) => {
  const tasks = await getUserTasks(userId);
  const timeTracking = await getTimeTracking(userId);
  
  const response = await fetch('http://localhost:8000/api/ml/burnout-analysis', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      avg_hours: timeTracking.avgDailyHours,
      task_count: tasks.active.length,
      overdue_count: tasks.overdue.length,
      stress_score: calculateStressScore(tasks)
    })
  });
  
  return await response.json();
};
```

### Predictive Project Insights

```python
@app.post("/api/ml/project-prediction")
async def predict_project_delay(project_data: ProjectData):
    """
    Predicts likelihood of project delay
    """
    features = prepare_project_features(project_data)
    delay_probability = project_model.predict_proba(features)
    
    return {
        "delay_probability": delay_probability,
        "estimated_delay_days": 5,
        "risk_factors": ["resource_shortage", "dependency_delays"]
    }
```

## Frontend Integration Patterns

### React Component for Task Board

```javascript
// components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inprogress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/tasks/user/${userId}`,
      {
        headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
      }
    );
    
    const grouped = response.data.reduce((acc, task) => {
      acc[task.status].push(task);
      return acc;
    }, { todo: [], inprogress: [], done: [] });
    
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await axios.patch(
      `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
      { status: newStatus },
      {
        headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
      }
    );
    
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {Object.entries(tasks).map(([status, taskList]) => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {taskList.map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status}
                onChange={(e) => updateTaskStatus(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="inprogress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default TaskBoard;
```

### Time Tracking Component

```javascript
// components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setElapsedTime(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const saveTime = async () => {
    await axios.post(
      `${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`,
      { duration: elapsedTime },
      {
        headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
      }
    );
    
    setElapsedTime(0);
    setIsRunning(false);
  };

  const formatTime = (seconds) => {
    const h = Math.floor(seconds / 3600);
    const m = Math.floor((seconds % 3600) / 60);
    const s = seconds % 60;
    return `${h}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(elapsedTime)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={saveTime} disabled={elapsedTime === 0}>
        Save Time
      </button>
    </div>
  );
};

export default TimeTracker;
```

### AI-Powered Ticket Form

```javascript
// components/TicketForm.jsx
import React, { useState } from 'react';
import axios from 'axios';

const TicketForm = () => {
  const [formData, setFormData] = useState({
    subject: '',
    description: '',
    priority: '',
    category: ''
  });
  const [aiSuggestions, setAiSuggestions] = useState(null);

  const handleInputChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const getAISuggestions = async () => {
    if (formData.subject && formData.description) {
      const response = await axios.post(
        `${process.env.REACT_APP_ML_URL}/api/ml/classify-ticket`,
        {
          subject: formData.subject,
          description: formData.description
        }
      );
      
      setAiSuggestions(response.data);
      setFormData({
        ...formData,
        priority: response.data.priority,
        category: response.data.category
      });
    }
  };

  const submitTicket = async (e) => {
    e.preventDefault();
    
    await axios.post(
      `${process.env.REACT_APP_API_URL}/tickets`,
      formData,
      {
        headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
      }
    );
    
    // Reset form
    setFormData({ subject: '', description: '', priority: '', category: '' });
    setAiSuggestions(null);
  };

  return (
    <form onSubmit={submitTicket}>
      <input
        name="subject"
        placeholder="Subject"
        value={formData.subject}
        onChange={handleInputChange}
        required
      />
      
      <textarea
        name="description"
        placeholder="Description"
        value={formData.description}
        onChange={handleInputChange}
        required
      />
      
      <button type="button" onClick={getAISuggestions}>
        Get AI Suggestions
      </button>
      
      {aiSuggestions && (
        <div className="ai-suggestions">
          <p>Suggested Priority: {aiSuggestions.priority}</p>
          <p>Suggested Category: {aiSuggestions.category}</p>
          <p>Confidence: {(aiSuggestions.confidence * 100).toFixed(0)}%</p>
        </div>
      )}
      
      <select 
        name="priority" 
        value={formData.priority} 
        onChange={handleInputChange}
        required
      >
        <option value="">Select Priority</option>
        <option value="low">Low</option>
        <option value="medium">Medium</option>
        <option value="high">High</option>
      </select>
      
      <select 
        name="category" 
        value={formData.category} 
        onChange={handleInputChange}
        required
      >
        <option value="">Select Category</option>
        <option value="technical">Technical</option>
        <option value="billing">Billing</option>
        <option value="general">General</option>
      </select>
      
      <button type="submit">Submit Ticket</button>
    </form>
  );
};

export default TicketForm;
```

## Backend Implementation Patterns

### User Model (MongoDB Schema)

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  isActive: {
    type: Boolean,
    default: true
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

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// models/Task.js
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
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  status: {
    type: String,
    enum: ['todo', 'inprogress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  dueDate: Date,
  timeTracked: {
    type: Number,
    default: 0  // in seconds
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      throw new Error();
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);
    
    if (!user || !user.isActive) {
      throw new Error();
    }
    
    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
  }
};

const adminAuth = async (req, res, next) => {
  await auth(req, res, () => {
    if (req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Admin access required' });
    }
    next();
  });
};

module.exports = { auth, adminAuth };
```

### Task Controller

```javascript
// controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id
    });
    
    await task.save();
    
    // Check for burnout risk when assigning tasks
    const burnoutCheck = await axios.post(
      `${process.env.ML_SERVICE_URL}/api/ml/burnout-analysis`,
      {
        userId: task.assignedTo,
        task_count: await Task.countDocuments({ 
          assignedTo: task.assignedTo, 
          status: { $ne: 'done' } 
        })
      }
    );
    
    res.status(201).json({ 
      task, 
      burnoutWarning: burnoutCheck.data.burnout_risk === 'high' 
    });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findById(req.params.taskId);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    task.status = req.body.status;
    
    if (req.body.status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const task = await Task.findById(req.params.taskId);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    task.timeTracked += req.body.duration;
    await task.save();
    
    res.json({ timeTracked: task.timeTracked });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

## ML Service Implementation

### Ticket Classifier Model

```python
# ml-service/models/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
import pickle
import os

class TicketClassifier:
    def __init__(self, model_path='./models'):
        self.model_path = model_path
        self.category_model = None
        self.priority_model = None
        
        if os.path.exists(f'{model_path}/category_model.pkl'):
            self.load_models()
        else:
            self.train_initial_models()
    
    def train_initial_models(self):
        """Train initial models with sample data"""
        # Category classifier
        self.category_model = Pipeline([
            ('tfidf', TfidfVectorizer(max_features=1000)),
            ('clf', MultinomialNB())
        ])
        
        # Priority classifier
        self.priority_model = Pipeline([
            ('tfidf', TfidfVectorizer(max_features=1000)),
            ('clf', MultinomialNB())
        ])
        
        # Sample training data (replace with actual data)
        sample_texts = [
            "login not working",
            "payment failed",
            "how to reset password",
            "urgent server down"
        ]
        sample_categories = ['technical', 'billing', 'general', 'technical']
        sample_priorities = ['medium', 'high', 'low', 'high']
        
        self.category_model.fit(sample_texts, sample_categories)
        self.priority_model.fit(sample_texts, sample_priorities)
        
        self.save_models()
    
    def predict_category(self, text):
        """Predict ticket category"""
        return self.category_model.predict([text])[0]
    
    def predict_priority(self, text):
        """Predict ticket priority"""
        return self.priority_model.predict([text])[0]
    
    def get_confidence(self, text, model_type='category'):
        """Get prediction confidence"""
        model = self.category_model if model_type == 'category' else self.priority_model
        proba = model.predict_proba([text])[0]
        return max(proba)
    
    def save_models(self):
        """Save trained models"""
        os.makedirs(self.model_path, exist_ok=True)
        
        with open(f'{self.model_path}/category_model.pkl', 'wb') as f:
            pickle.dump(self.category_model, f)
        
        with open(f'{self.model_path}/priority_model.pkl', 'wb') as f:
            pickle.dump(self.priority_model, f)
    
    def load_models(self):
        """Load saved models"""
        with open(f'{self.model_path}/category_model.pkl', 'rb') as f:
            self.category_model = pickle.load(f)
        
        with open(f'{self.model_path}/priority_model.pkl', 'rb') as f:
            self.priority_model = pickle.load(f)
```

### Risk Detection Model

```python
# ml-service/models/risk_detector.py
from river import anomaly
from river import preprocessing
import numpy as np

class RiskDetector:
    def __init__(self):
        self.model = preprocessing.StandardScaler() | anomaly.HalfSpaceTrees(
            n_trees=10,
            height=8,
            window_size=250
        )
        self.feature_names = [
            'login_hour',
            'failed_attempts',
            'session_duration',
            'unusual_ip',
            'data_access_volume'
        ]
    
    def extract_features(self, user_behavior):
        """Extract features from user behavior"""
        features = {
            'login_hour': user_behavior.get('login_hour', 12),
            'failed_attempts': user_behavior.get('failed_attempts', 0),
            'session_duration': user_behavior.get('session_duration', 0),
            'unusual_ip': 1 if user_behavior.get('unusual_ip', False) else 0,
            'data_access_volume': user_behavior.get('data_access_volume', 0)
        }
        return features
    
    def predict_risk(self, user_behavior):
        """Predict risk score for user behavior"""
        features = self.extract_features(user_behavior)
        
        # Get anomaly score
        score = self.model.score_one(features)
        
        # Update model (online learning)
        self.model.learn_one(features)
        
        # Normalize score to 0-1 range
        risk_score = min(1.0, max(0.0, score / 2.0))
        
        # Determine risk level
        if risk_score > 0.7:
            risk_level = 'high'
        elif risk_score > 0.4:
            risk_level = 'medium'
        else:
            risk_level = 'low'
        
        # Identify risk factors
        risk_factors = []
        if features['failed_attempts'] > 3:
            risk_factors.append('high_failed_attempts')
        if features['unusual_ip']:
            risk_factors.append('unusual_ip')
        if features['login_hour'] < 6 or features['login_hour'] > 22:
            risk_factors.append('unusual_login_time')
        
        return {
            'risk_level': risk_level,
            'risk_score': risk_score,
            'factors': risk_factors
        }
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List
from models.ticket_classifier import TicketClassifier
from models.risk_detector import RiskDetector
import uvicorn

app = FastAPI(title="Enterprise UMS ML Service")

# Initialize models
ticket_classifier = TicketClassifier()
risk_detector = RiskDetector()

# Request models
class TicketInput(BaseModel):
    subject: str
    description: str

class UserBehavior(BaseModel):
    userId: str
    login_hour: Optional[int] = 12
    failed_attempts: Optional[int] = 0
    session_duration: Optional[float] = 0
    unusual_ip: Optional[bool] = False
    data_access_volume: Optional[int] = 0

class WorkloadData(BaseModel):
    userId: str
    avg_hours: float
    task_count: int
    overdue_count: int
    stress_score: Optional[float] = 0

# Endpoints
@app.post("/api/ml/classify-ticket")
async def classify_ticket(ticket: TicketInput):
    """Classify ticket category and priority"""
    try:
        combined_text = f"{ticket.subject} {ticket.description}"
        
        category = ticket_classifier.predict_category(combined_text)
        priority = ticket_classifier.predict_priority(combined_text)
        confidence = ticket_classifier.get_confidence(combined_text)
        
        return {
            "category": category,
            "priority": priority,
            "confidence": float(confidence)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.
