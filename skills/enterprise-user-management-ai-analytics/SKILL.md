---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered risk detection, anomaly analysis, and task management using React, Node.js, and FastAPI ML services
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics into user management
  - implement JWT authentication for admin and user roles
  - create AI-powered ticket classification system
  - build task management with burnout detection
  - set up ML service for risk prediction
  - configure MongoDB for user management app
  - implement Kanban board with time tracking
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines traditional user management with AI-powered insights. It provides role-based access control (Admin/User), task management with Kanban boards, support ticket system, and ML-driven analytics for risk detection, anomaly detection, burnout analysis, and predictive insights.

**Key Components:**
- **Frontend**: React.js with JWT authentication
- **Backend**: Node.js REST API with MongoDB
- **ML Service**: FastAPI with scikit-learn and River for online learning
- **Features**: User CRUD, task tracking, ticket management, AI analytics

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
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt

# Install frontend dependencies
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env)**
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**
```bash
# ml-service/.env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
MODEL_PATH=./models
LOG_LEVEL=INFO
```

**Frontend (.env)**
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

## Running the Application

```bash
# Terminal 1 - Backend
cd backend
npm start
# Runs on http://localhost:5000

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload
# Runs on http://localhost:8000

# Terminal 3 - Frontend
cd frontend
npm start
# Runs on http://localhost:3000
```

## Backend API Structure

### User Authentication

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    // Check if user exists
    let user = await User.findOne({ email });
    if (user) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);
    
    // Create user
    user = new User({
      name,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    // Generate JWT
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.status(201).json({
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
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
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );
    
    res.json({
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const auth = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No token, authorization denied' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Token is not valid' });
  }
};

const adminAuth = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Access denied. Admin only.' });
  }
  next();
};

module.exports = { auth, adminAuth };
```

### Task Management

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  status: { type: String, enum: ['todo', 'inprogress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: { type: Date },
  timeSpent: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', TaskSchema);
```

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { auth, adminAuth } = require('../middleware/auth');

// Create task (Admin only)
router.post('/', auth, adminAuth, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      assignedBy: req.user.id
    });
    await task.save();
    await task.populate(['assignedTo', 'assignedBy']);
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Get user tasks
router.get('/my-tasks', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'name email')
      .sort('-createdAt');
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', auth, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    // Users can only update their own tasks
    if (task.assignedTo.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Access denied' });
    }
    
    task.status = status;
    task.updatedAt = Date.now();
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Update time spent
router.patch('/:id/time', auth, async (req, res) => {
  try {
    const { timeSpent } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeSpent }, updatedAt: Date.now() },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

module.exports = router;
```

### Support Ticket System

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const TicketSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  category: { type: String, enum: ['technical', 'billing', 'general', 'urgent'], default: 'general' },
  priority: { type: String, enum: ['low', 'medium', 'high', 'critical'], default: 'medium' },
  status: { type: String, enum: ['open', 'in_progress', 'resolved', 'closed'], default: 'open' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  aiClassification: { type: String },
  aiRiskScore: { type: Number },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', TicketSchema);
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
    
    // Call ML service for classification
    let aiClassification = null;
    let aiRiskScore = null;
    
    try {
      const mlResponse = await axios.post(`${process.env.ML_SERVICE_URL}/classify-ticket`, {
        title,
        description
      });
      aiClassification = mlResponse.data.category;
      aiRiskScore = mlResponse.data.risk_score;
    } catch (mlError) {
      console.error('ML service error:', mlError.message);
    }
    
    const ticket = new Ticket({
      title,
      description,
      category: category || aiClassification || 'general',
      createdBy: req.user.id,
      aiClassification,
      aiRiskScore
    });
    
    await ticket.save();
    await ticket.populate('createdBy', 'name email');
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Get all tickets (Admin)
router.get('/', auth, adminAuth, async (req, res) => {
  try {
    const tickets = await Ticket.find()
      .populate('createdBy', 'name email')
      .populate('assignedTo', 'name email')
      .sort('-createdAt');
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', auth, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.id })
      .populate('assignedTo', 'name email')
      .sort('-createdAt');
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI Setup

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from typing import Optional, List
import joblib
import os
from datetime import datetime

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models directory
MODEL_DIR = os.getenv("MODEL_PATH", "./models")
os.makedirs(MODEL_DIR, exist_ok=True)

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_logins: int
    unusual_hours_activity: int
    data_access_violations: int
    policy_violations: int

class BurnoutAnalysisRequest(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_task_time: float
    overtime_hours: float
    ticket_count: int

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    login_time: str
    ip_address: str
    location: str
    device: str
    
@app.get("/")
def read_root():
    return {"status": "ML Service is running", "version": "1.0.0"}
```

### Ticket Classification

```python
# ml-service/services/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
import numpy as np

class TicketClassifier:
    def __init__(self):
        self.model = Pipeline([
            ('tfidf', TfidfVectorizer(max_features=1000, stop_words='english')),
            ('classifier', MultinomialNB())
        ])
        self.categories = ['technical', 'billing', 'general', 'urgent']
        self.trained = False
    
    def train(self, texts, labels):
        """Train the classifier with sample data"""
        self.model.fit(texts, labels)
        self.trained = True
    
    def predict(self, text):
        """Predict category for new ticket"""
        if not self.trained:
            # Return default if not trained
            return 'general', 0.5
        
        prediction = self.model.predict([text])[0]
        probabilities = self.model.predict_proba([text])[0]
        confidence = np.max(probabilities)
        
        return prediction, confidence
    
    def calculate_risk_score(self, text, category):
        """Calculate risk score based on text content"""
        urgent_keywords = ['urgent', 'critical', 'emergency', 'asap', 'immediately', 
                          'down', 'broken', 'not working', 'crash', 'error']
        
        text_lower = text.lower()
        keyword_count = sum(1 for keyword in urgent_keywords if keyword in text_lower)
        
        # Base risk by category
        category_risk = {
            'urgent': 0.8,
            'technical': 0.6,
            'billing': 0.5,
            'general': 0.3
        }
        
        base_risk = category_risk.get(category, 0.3)
        keyword_risk = min(keyword_count * 0.1, 0.5)
        
        total_risk = min(base_risk + keyword_risk, 1.0)
        return round(total_risk, 2)

# Initialize classifier
ticket_classifier = TicketClassifier()

# Train with sample data
sample_texts = [
    "System is down and not responding",
    "Unable to access my account",
    "Need invoice for last month",
    "Application keeps crashing",
    "How do I reset my password",
    "URGENT: Server not accessible",
    "Payment not processed",
    "General inquiry about features"
]
sample_labels = ['urgent', 'technical', 'billing', 'technical', 
                'general', 'urgent', 'billing', 'general']

ticket_classifier.train(sample_texts, sample_labels)
```

```python
# ml-service/main.py (continued)
from services.ticket_classifier import ticket_classifier

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        combined_text = f"{request.title} {request.description}"
        category, confidence = ticket_classifier.predict(combined_text)
        risk_score = ticket_classifier.calculate_risk_score(
            combined_text, 
            category
        )
        
        return {
            "category": category,
            "confidence": float(confidence),
            "risk_score": risk_score,
            "recommended_priority": "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Risk Prediction

```python
# ml-service/services/risk_predictor.py
import numpy as np
from sklearn.ensemble import RandomForestClassifier

class RiskPredictor:
    def __init__(self):
        self.model = RandomForestClassifier(n_estimators=100, random_state=42)
        self.trained = False
        self._train_initial_model()
    
    def _train_initial_model(self):
        """Train with synthetic data"""
        # Generate synthetic training data
        np.random.seed(42)
        n_samples = 1000
        
        X = np.random.rand(n_samples, 4)
        # failed_logins, unusual_hours, data_violations, policy_violations
        X[:, 0] *= 10  # failed logins 0-10
        X[:, 1] *= 20  # unusual hours 0-20
        X[:, 2] *= 5   # data violations 0-5
        X[:, 3] *= 5   # policy violations 0-5
        
        # Risk logic: high values = high risk
        risk_score = (X[:, 0] * 0.3 + X[:, 1] * 0.2 + 
                     X[:, 2] * 0.3 + X[:, 3] * 0.2)
        y = (risk_score > 5).astype(int)
        
        self.model.fit(X, y)
        self.trained = True
    
    def predict_risk(self, failed_logins, unusual_hours, 
                    data_violations, policy_violations):
        """Predict risk level for a user"""
        features = np.array([[
            failed_logins,
            unusual_hours,
            data_violations,
            policy_violations
        ]])
        
        risk_prob = self.model.predict_proba(features)[0][1]
        risk_level = "high" if risk_prob > 0.7 else "medium" if risk_prob > 0.4 else "low"
        
        # Calculate detailed risk score
        risk_score = (
            failed_logins * 0.3 +
            unusual_hours * 0.2 +
            data_violations * 0.3 +
            policy_violations * 0.2
        ) / 10.0
        
        return {
            "risk_probability": round(float(risk_prob), 2),
            "risk_level": risk_level,
            "risk_score": round(min(risk_score, 1.0), 2),
            "factors": {
                "failed_logins": failed_logins,
                "unusual_hours_activity": unusual_hours,
                "data_access_violations": data_violations,
                "policy_violations": policy_violations
            }
        }

risk_predictor = RiskPredictor()
```

```python
# ml-service/main.py (continued)
from services.risk_predictor import risk_predictor

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        result = risk_predictor.predict_risk(
            request.failed_logins,
            request.unusual_hours_activity,
            request.data_access_violations,
            request.policy_violations
        )
        result["user_id"] = request.user_id
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Burnout Detection

```python
# ml-service/services/burnout_detector.py
import numpy as np

class BurnoutDetector:
    def __init__(self):
        self.thresholds = {
            "task_overload": 0.7,  # completion rate < 70%
            "overtime_critical": 20,  # hours per week
            "avg_time_high": 4.0,  # hours per task
            "ticket_overload": 10  # support tickets
        }
    
    def analyze_burnout(self, tasks_assigned, tasks_completed, 
                       avg_task_time, overtime_hours, ticket_count):
        """Analyze burnout risk for a user"""
        
        # Calculate metrics
        completion_rate = tasks_completed / max(tasks_assigned, 1)
        
        # Burnout indicators
        indicators = []
        score = 0.0
        
        # Task completion rate
        if completion_rate < self.thresholds["task_overload"]:
            indicators.append({
                "indicator": "Low task completion rate",
                "value": f"{completion_rate * 100:.1f}%",
                "severity": "high" if completion_rate < 0.5 else "medium"
            })
            score += 0.3
        
        # Overtime hours
        if overtime_hours > self.thresholds["overtime_critical"]:
            indicators.append({
                "indicator": "Excessive overtime",
                "value": f"{overtime_hours} hours/week",
                "severity": "high"
            })
            score += 0.3
        
        # Average task time
        if avg_task_time > self.thresholds["avg_time_high"]:
            indicators.append({
                "indicator": "High average task time",
                "value": f"{avg_task_time:.1f} hours",
                "severity": "medium"
            })
            score += 0.2
        
        # Support ticket overload
        if ticket_count > self.thresholds["ticket_overload"]:
            indicators.append({
                "indicator": "High support ticket volume",
                "value": f"{ticket_count} tickets",
                "severity": "medium"
            })
            score += 0.2
        
        burnout_risk = min(score, 1.0)
        risk_level = "high" if burnout_risk > 0.6 else "medium" if burnout_risk > 0.3 else "low"
        
        return {
            "burnout_risk_score": round(burnout_risk, 2),
            "risk_level": risk_level,
            "indicators": indicators,
            "recommendations": self._get_recommendations(risk_level, indicators),
            "metrics": {
                "completion_rate": round(completion_rate, 2),
                "overtime_hours": overtime_hours,
                "avg_task_time": avg_task_time,
                "ticket_count": ticket_count
            }
        }
    
    def _get_recommendations(self, risk_level, indicators):
        """Generate recommendations based on risk level"""
        if risk_level == "low":
            return ["Workload is manageable", "Continue current pace"]
        
        recommendations = []
        
        for indicator in indicators:
            if "completion rate" in indicator["indicator"]:
                recommendations.append("Redistribute tasks to balance workload")
            elif "overtime" in indicator["indicator"]:
                recommendations.append("Reduce overtime hours immediately")
            elif "task time" in indicator["indicator"]:
                recommendations.append("Review task complexity and provide support")
            elif "ticket" in indicator["indicator"]:
                recommendations.append("Assign additional support for ticket handling")
        
        if risk_level == "high":
            recommendations.append("Consider immediate workload reduction")
            recommendations.append("Schedule wellness check-in")
        
        return recommendations

burnout_detector = BurnoutDetector()
```

```python
# ml-service/main.py (continued)
from services.burnout_detector import burnout_detector

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    try:
        result = burnout_detector.analyze_burnout(
            request.tasks_assigned,
            request.tasks_completed,
            request.avg_task_time,
            request.overtime_hours,
            request.ticket_count
        )
        result["user_id"] = request.user_id
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
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
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(res.data);
    } catch (error) {
      console.error('Auth error:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    const { token, user } = res.data;
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
    delete axios.defaults.headers.common['Authorization'];
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Task Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/tasks/my-tasks`);
      const grouped = res.data.reduce((acc, task) => {
        acc[task.status].push(task);
        return acc;
      }, { todo: [], inprogress: [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = (e, status) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, status);
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const TaskCard = ({ task }) => (
    <div
      className="task-card"
      draggable
      onDragStart={(e) => handleDragStart(e, task._id)}
    >
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <div className="task-meta">
        <span className={`priority ${task.priority}`}>{task.priority}</span>
        <span className="time">{task.timeSpent} min</span>
      </div>
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      <div
        className="kanban-column"
        onDrop={(e) => handleDrop(e, 'todo')}
        onDragOver={handleDragOver}
      >
        <h3>To Do ({tasks.todo.length})</h3>
        <div className="task-list">
          {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
        
