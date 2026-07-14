---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, risk prediction, and burnout detection
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement user task management with AI insights"
  - "create admin dashboard with user analytics"
  - "build ticket classification system with ML"
  - "add risk prediction and anomaly detection"
  - "integrate AI assistant for user management"
  - "configure kanban board with time tracking"
  - "deploy user management system with FastAPI ML service"
---

# Enterprise User Management AI Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

Enterprise User Management System with AI Analytics is a full-stack application that combines user/task management with machine learning capabilities. It provides:

- **User & Admin Dashboards**: Role-based access for task management and system administration
- **AI-Powered Insights**: Risk prediction, anomaly detection, burnout analysis, and predictive project insights
- **Smart Ticket System**: ML-based ticket classification and intelligent routing
- **Task Management**: Kanban boards, time tracking, and progress monitoring
- **Real-time Analytics**: Organization-wide metrics and performance dashboards

The system consists of three main components:
1. **Frontend**: React.js application
2. **Backend**: Node.js REST API with MongoDB
3. **ML Service**: FastAPI service with scikit-learn and River for online learning

## Installation

### Prerequisites

```bash
# Required
- Node.js 14+ and npm
- Python 3.8+
- MongoDB running locally or connection URI
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
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# ML service runs at http://localhost:8000
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
# Frontend runs at http://localhost:3000
```

## Backend API Structure

### Authentication Endpoints

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

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
      { expiresIn: process.env.JWT_EXPIRE }
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
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
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
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Middleware for Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No token provided' });
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
const { authMiddleware } = require('../middleware/auth');

// Get user tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('assignedBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json({ tasks });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority, dueDate, assignedTo } = req.body;
    
    const task = new Task({
      title,
      description,
      priority: priority || 'medium',
      dueDate,
      assignedTo,
      assignedBy: req.user.userId,
      status: 'todo'
    });
    
    await task.save();
    await task.populate('assignedTo assignedBy', 'username email');
    
    res.status(201).json({ task });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.json({ task });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Track time
router.post('/:id/time-log', authMiddleware, async (req, res) => {
  try {
    const { duration, note } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    task.timeTracking.push({
      userId: req.user.userId,
      duration,
      note,
      timestamp: new Date()
    });
    
    await task.save();
    res.json({ task });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Support Ticket API

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const Ticket = require('../models/Ticket');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Get AI classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      { title, description }
    );
    
    const ticket = new Ticket({
      title,
      description,
      priority: priority || 'medium',
      createdBy: req.user.userId,
      category: mlResponse.data.category,
      assignedTo: mlResponse.data.suggestedAssignee,
      status: 'open'
    });
    
    await ticket.save();
    await ticket.populate('createdBy assignedTo', 'username email');
    
    res.status(201).json({ ticket });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Get user tickets
router.get('/', authMiddleware, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.userId };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json({ tickets });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update ticket status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status, resolution } = req.body;
    const ticket = await Ticket.findById(req.params.id);
    
    if (!ticket) {
      return res.status(404).json({ message: 'Ticket not found' });
    }
    
    ticket.status = status;
    if (resolution) ticket.resolution = resolution;
    if (status === 'closed') ticket.closedAt = new Date();
    
    await ticket.save();
    res.json({ ticket });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from datetime import datetime
import joblib
import os
from models.ticket_classifier import TicketClassifier
from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector
from models.anomaly_detector import AnomalyDetector

app = FastAPI(title="Enterprise AI Analytics Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize models
ticket_classifier = TicketClassifier()
risk_predictor = RiskPredictor()
burnout_detector = BurnoutDetector()
anomaly_detector = AnomalyDetector()

# Request models
class TicketRequest(BaseModel):
    title: str
    description: str

class RiskAnalysisRequest(BaseModel):
    userId: str
    tasksCompleted: int
    tasksOverdue: int
    avgResponseTime: float
    ticketsRaised: int

class BurnoutRequest(BaseModel):
    userId: str
    workHoursPerWeek: float
    tasksCount: int
    overtimeHours: float
    lastBreakDays: int

class AnomalyRequest(BaseModel):
    userId: str
    loginTime: str
    loginLocation: str
    activityPattern: List[float]

@app.get("/")
async def root():
    return {"message": "Enterprise AI Analytics Service", "status": "running"}

@app.post("/classify-ticket")
async def classify_ticket(request: TicketRequest):
    """Classify ticket and suggest assignee"""
    try:
        category = ticket_classifier.predict(request.title, request.description)
        suggested_assignee = ticket_classifier.get_best_assignee(category)
        confidence = ticket_classifier.get_confidence()
        
        return {
            "category": category,
            "suggestedAssignee": suggested_assignee,
            "confidence": confidence
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict-risk")
async def predict_risk(request: RiskAnalysisRequest):
    """Predict user risk level"""
    try:
        features = np.array([
            request.tasksCompleted,
            request.tasksOverdue,
            request.avgResponseTime,
            request.ticketsRaised
        ]).reshape(1, -1)
        
        risk_level = risk_predictor.predict(features)
        risk_score = risk_predictor.get_risk_score(features)
        factors = risk_predictor.get_risk_factors(request)
        
        return {
            "userId": request.userId,
            "riskLevel": risk_level,
            "riskScore": float(risk_score),
            "factors": factors,
            "recommendations": risk_predictor.get_recommendations(risk_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-burnout")
async def detect_burnout(request: BurnoutRequest):
    """Detect employee burnout"""
    try:
        features = np.array([
            request.workHoursPerWeek,
            request.tasksCount,
            request.overtimeHours,
            request.lastBreakDays
        ]).reshape(1, -1)
        
        burnout_risk = burnout_detector.predict(features)
        burnout_score = burnout_detector.get_score(features)
        
        return {
            "userId": request.userId,
            "burnoutRisk": burnout_risk,
            "burnoutScore": float(burnout_score),
            "recommendations": burnout_detector.get_recommendations(burnout_risk),
            "warningLevel": burnout_detector.get_warning_level(burnout_score)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyRequest):
    """Detect anomalous user behavior"""
    try:
        is_anomaly = anomaly_detector.predict(
            request.userId,
            request.loginTime,
            request.loginLocation,
            request.activityPattern
        )
        
        anomaly_score = anomaly_detector.get_anomaly_score()
        details = anomaly_detector.get_anomaly_details()
        
        return {
            "userId": request.userId,
            "isAnomaly": bool(is_anomaly),
            "anomalyScore": float(anomaly_score),
            "details": details,
            "alertLevel": "high" if is_anomaly else "normal"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/train-ticket-classifier")
async def train_ticket_classifier(tickets: List[dict]):
    """Retrain ticket classifier with new data"""
    try:
        ticket_classifier.train(tickets)
        return {"message": "Model trained successfully", "samplesProcessed": len(tickets)}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "models": {
            "ticketClassifier": "loaded",
            "riskPredictor": "loaded",
            "burnoutDetector": "loaded",
            "anomalyDetector": "loaded"
        }
    }
```

### Ticket Classifier Model

```python
# ml-service/models/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
import numpy as np
import joblib
import os

class TicketClassifier:
    def __init__(self):
        self.model = Pipeline([
            ('tfidf', TfidfVectorizer(max_features=1000, stop_words='english')),
            ('classifier', MultinomialNB())
        ])
        self.categories = ['technical', 'billing', 'access', 'feature_request', 'bug_report']
        self.assignee_map = {
            'technical': 'tech_support_team',
            'billing': 'finance_team',
            'access': 'admin_team',
            'feature_request': 'product_team',
            'bug_report': 'dev_team'
        }
        self.confidence = 0.0
        self.load_model()
    
    def load_model(self):
        model_path = os.getenv('MODEL_PATH', './models')
        try:
            self.model = joblib.load(f"{model_path}/ticket_classifier.pkl")
        except FileNotFoundError:
            # Train with default data if model doesn't exist
            self._train_default()
    
    def _train_default(self):
        # Default training data
        texts = [
            "Cannot login to system",
            "Payment not processed",
            "Need new feature for reporting",
            "Application crashes on startup",
            "Invoice is incorrect"
        ]
        labels = ['access', 'billing', 'feature_request', 'bug_report', 'billing']
        
        self.model.fit(texts, labels)
        self._save_model()
    
    def predict(self, title: str, description: str) -> str:
        text = f"{title} {description}"
        prediction = self.model.predict([text])[0]
        
        # Get prediction probabilities for confidence
        if hasattr(self.model.named_steps['classifier'], 'predict_proba'):
            proba = self.model.named_steps['classifier'].predict_proba(
                self.model.named_steps['tfidf'].transform([text])
            )[0]
            self.confidence = float(max(proba))
        
        return prediction
    
    def get_best_assignee(self, category: str) -> str:
        return self.assignee_map.get(category, 'general_support')
    
    def get_confidence(self) -> float:
        return self.confidence
    
    def train(self, tickets: list):
        texts = [f"{t['title']} {t['description']}" for t in tickets]
        labels = [t['category'] for t in tickets]
        
        self.model.fit(texts, labels)
        self._save_model()
    
    def _save_model(self):
        model_path = os.getenv('MODEL_PATH', './models')
        os.makedirs(model_path, exist_ok=True)
        joblib.dump(self.model, f"{model_path}/ticket_classifier.pkl")
```

### Risk Predictor Model

```python
# ml-service/models/risk_predictor.py
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import tree, ensemble
import joblib
import os

class RiskPredictor:
    def __init__(self):
        # Use River for online learning
        self.model = ensemble.AdaptiveRandomForestClassifier(
            n_models=10,
            seed=42
        )
        self.threshold_high = 0.7
        self.threshold_medium = 0.4
        self.load_model()
    
    def load_model(self):
        model_path = os.getenv('MODEL_PATH', './models')
        try:
            self.model = joblib.load(f"{model_path}/risk_predictor.pkl")
        except FileNotFoundError:
            pass  # Use default model
    
    def predict(self, features: np.ndarray) -> str:
        """Predict risk level: low, medium, high"""
        score = self.get_risk_score(features)
        
        if score >= self.threshold_high:
            return "high"
        elif score >= self.threshold_medium:
            return "medium"
        else:
            return "low"
    
    def get_risk_score(self, features: np.ndarray) -> float:
        """Calculate risk score from features"""
        tasks_completed, tasks_overdue, avg_response_time, tickets_raised = features[0]
        
        # Normalize features
        overdue_ratio = tasks_overdue / (tasks_completed + 1)
        response_penalty = min(avg_response_time / 24, 1)  # Normalize to 24 hours
        ticket_penalty = min(tickets_raised / 10, 1)  # Normalize to 10 tickets
        
        risk_score = (
            overdue_ratio * 0.4 +
            response_penalty * 0.3 +
            ticket_penalty * 0.3
        )
        
        return min(risk_score, 1.0)
    
    def get_risk_factors(self, request) -> list:
        factors = []
        
        if request.tasksOverdue > 3:
            factors.append("High number of overdue tasks")
        if request.avgResponseTime > 12:
            factors.append("Slow response time")
        if request.ticketsRaised > 5:
            factors.append("Multiple support tickets raised")
        
        return factors if factors else ["No significant risk factors detected"]
    
    def get_recommendations(self, risk_level: str) -> list:
        recommendations = {
            "high": [
                "Immediate manager intervention required",
                "Redistribute workload",
                "Schedule one-on-one meeting",
                "Review task priorities"
            ],
            "medium": [
                "Monitor closely",
                "Provide additional support",
                "Review upcoming deadlines"
            ],
            "low": [
                "Continue current performance",
                "Regular check-ins"
            ]
        }
        return recommendations.get(risk_level, [])
```

### Burnout Detector

```python
# ml-service/models/burnout_detector.py
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier

class BurnoutDetector:
    def __init__(self):
        self.thresholds = {
            'work_hours': 50,
            'overtime': 10,
            'last_break': 30
        }
    
    def predict(self, features: np.ndarray) -> str:
        """Predict burnout risk: low, moderate, high, critical"""
        score = self.get_score(features)
        
        if score >= 0.75:
            return "critical"
        elif score >= 0.5:
            return "high"
        elif score >= 0.3:
            return "moderate"
        else:
            return "low"
    
    def get_score(self, features: np.ndarray) -> float:
        work_hours, tasks_count, overtime, last_break = features[0]
        
        # Calculate normalized scores
        work_score = min(work_hours / 60, 1.0)  # Normalize to 60 hours
        task_score = min(tasks_count / 20, 1.0)  # Normalize to 20 tasks
        overtime_score = min(overtime / 15, 1.0)  # Normalize to 15 hours
        break_score = min(last_break / 60, 1.0)  # Normalize to 60 days
        
        burnout_score = (
            work_score * 0.3 +
            task_score * 0.2 +
            overtime_score * 0.3 +
            break_score * 0.2
        )
        
        return burnout_score
    
    def get_recommendations(self, burnout_risk: str) -> list:
        recommendations = {
            "critical": [
                "Immediate action required",
                "Mandatory time off",
                "Reduce workload by 50%",
                "Schedule wellness consultation"
            ],
            "high": [
                "Reduce overtime",
                "Schedule break days",
                "Delegate tasks",
                "Weekly check-ins"
            ],
            "moderate": [
                "Monitor workload",
                "Encourage work-life balance",
                "Regular breaks recommended"
            ],
            "low": [
                "Maintain current balance",
                "Continue healthy work habits"
            ]
        }
        return recommendations.get(burnout_risk, [])
    
    def get_warning_level(self, score: float) -> str:
        if score >= 0.75:
            return "CRITICAL"
        elif score >= 0.5:
            return "WARNING"
        elif score >= 0.3:
            return "CAUTION"
        else:
            return "NORMAL"
```

## Frontend Integration

### React Hook for API Calls

```javascript
// frontend/src/hooks/useAPI.js
import { useState, useCallback } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

export const useAPI = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const getAuthHeaders = () => {
    const token = localStorage.getItem('token');
    return {
      headers: {
        Authorization: `Bearer ${token}`,
        'Content-Type': 'application/json'
      }
    };
  };

  const callAPI = useCallback(async (endpoint, method = 'GET', data = null, isML = false) => {
    setLoading(true);
    setError(null);

    try {
      const baseURL = isML ? ML_API_URL : API_URL;
      const config = {
        method,
        url: `${baseURL}${endpoint}`,
        ...getAuthHeaders()
      };

      if (data) {
        config.data = data;
      }

      const response = await axios(config);
      setLoading(false);
      return response.data;
    } catch (err) {
      setError(err.response?.data?.message || 'An error occurred');
      setLoading(false);
      throw err;
    }
  }, []);

  return { callAPI, loading, error };
};
```

### Task Dashboard Component

```javascript
// frontend/src/components/TaskDashboard.jsx
import React, { useState, useEffect } from 'react';
import { useAPI } from '../hooks/useAPI';
import './TaskDashboard.css';

const TaskDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [timeTracking, setTimeTracking] = useState({});
  const { callAPI, loading, error } = useAPI();

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await callAPI('/api/tasks');
      organizeTasks(response.tasks);
    } catch (err) {
      console.error('Failed to fetch tasks:', err);
    }
  };

  const organizeTasks = (taskList) => {
    const organized = {
      todo: taskList.filter(t => t.status === 'todo'),
      inProgress: taskList.filter(t => t.status === 'in_progress'),
      done: taskList.filter(t => t.status === 'done')
    };
    setTasks(organized);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await callAPI(`/api/tasks/${taskId}/status`, 'PATCH', { status: newStatus });
      fetchTasks();
    } catch (err) {
      console.error('Failed to update task:', err);
    }
  };

  const startTimer = (taskId) => {
    setTimeTracking(prev => ({
      ...prev,
      [taskId]: { startTime: Date.now(), duration: 0 }
    }));
  };

  const stopTimer = async (taskId) => {
    const tracking = timeTracking[taskId];
    if (!tracking) return;

    const duration = Math.floor((Date.now() - tracking.startTime) / 1000);
    
    try {
      await callAPI(`/api/tasks/${taskId}/time-log`, 'POST', {
        duration,
        note: 'Work session'
      });
      
      setTimeTracking(prev => {
        const updated = { ...prev };
        delete updated[taskId];
        return updated;
      });
      
      fetchTasks();
    } catch (err) {
      console.error('Failed to log time:', err);
    }
  };

  const TaskCard = ({ task, status }) => {
    const isTracking = timeTracking[task._id];
    
    return (
      <div className="task-card">
        <h4>{task.title}</h4>
        <p>{task.description}</p>
        <div className="task-meta">
          <span className={`priority ${task.priority}`}>{task.priority}</span>
          <span className="due-date">{new Date(task.dueDate).toLocaleDateString()}</span>
        </div>
        <div className="task-actions">
          {isTracking ? (
            <button
