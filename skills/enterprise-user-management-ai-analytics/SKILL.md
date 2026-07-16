---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "integrate ML service for anomaly detection"
  - "build admin panel with user management"
  - "configure JWT authentication for enterprise app"
  - "deploy user management system with AI features"
  - "troubleshoot FastAPI ML service integration"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build, configure, and extend the Enterprise User Management System with AI Analytics. The system is a full-stack JavaScript/Python application providing user management, task tracking, support ticketing, and AI-powered insights including risk detection, anomaly detection, burnout analysis, and predictive analytics.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Role-based access control, user CRUD operations, authentication via JWT
- **Task Management**: Kanban board (To Do → In Progress → Done), time tracking, assignment workflows
- **Support Tickets**: AI-powered classification, routing, and status tracking
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Admin analytics dashboard and user-specific performance insights

## Architecture Overview

The system consists of three main components:

1. **Frontend** (React.js) - Port 3000
2. **Backend** (Node.js) - Port 5000
3. **ML Service** (FastAPI + scikit-learn) - Port 8000

## Installation & Setup

### Prerequisites

```bash
# Node.js 14+ and Python 3.8+ required
node --version
python --version
```

### Clone and Install

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
MONGODB_URI=mongodb://localhost:27017/enterprise_management
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

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

npm start
```

## Backend API Patterns

### Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }
  
  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) return res.status(403).json({ error: 'Invalid token' });
    req.user = user;
    next();
  });
};

module.exports = { authenticateToken };
```

### User Management Routes

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { authenticateToken } = require('../middleware/auth');

// Get all users (Admin only)
router.get('/', authenticateToken, async (req, res) => {
  try {
    if (req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Admin access required' });
    }
    
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create user
router.post('/', authenticateToken, async (req, res) => {
  try {
    const { username, email, role, department } = req.body;
    
    const user = new User({
      username,
      email,
      role: role || 'user',
      department
    });
    
    await user.save();
    res.status(201).json(user);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update user
router.put('/:id', authenticateToken, async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authenticateToken } = require('../middleware/auth');

// Get tasks for current user
router.get('/my-tasks', authenticateToken, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId })
      .populate('createdBy', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authenticateToken, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      createdBy: req.user.userId,
      priority: priority || 'medium',
      status: 'todo',
      dueDate
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authenticateToken, async (req, res) => {
  try {
    const { status } = req.body;
    
    if (!['todo', 'in_progress', 'done'].includes(status)) {
      return res.status(400).json({ error: 'Invalid status' });
    }
    
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status },
      { new: true }
    );
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service API Patterns

### Risk Detection

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import IsolationForest
import joblib
import os

app = FastAPI()

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

class UserBehavior(BaseModel):
    user_id: str
    login_frequency: float
    task_completion_rate: float
    avg_session_duration: float
    failed_logins: int
    data_access_count: int

class RiskPrediction(BaseModel):
    user_id: str
    risk_score: float
    is_anomaly: bool
    factors: List[str]

@app.post("/api/ml/risk-detection", response_model=RiskPrediction)
async def detect_risk(behavior: UserBehavior):
    """Detect user risk based on behavior patterns"""
    try:
        # Extract features
        features = np.array([[
            behavior.login_frequency,
            behavior.task_completion_rate,
            behavior.avg_session_duration,
            behavior.failed_logins,
            behavior.data_access_count
        ]])
        
        # Load or train model
        model_path = os.path.join(MODEL_PATH, 'risk_detector.pkl')
        if os.path.exists(model_path):
            model = joblib.load(model_path)
        else:
            model = IsolationForest(contamination=0.1, random_state=42)
            # Train with initial data (in production, use historical data)
            model.fit(features)
            joblib.dump(model, model_path)
        
        # Predict
        prediction = model.predict(features)[0]
        risk_score = model.score_samples(features)[0]
        
        # Identify risk factors
        factors = []
        if behavior.failed_logins > 3:
            factors.append("High failed login attempts")
        if behavior.task_completion_rate < 0.5:
            factors.append("Low task completion rate")
        if behavior.data_access_count > 100:
            factors.append("Unusual data access pattern")
        
        return RiskPrediction(
            user_id=behavior.user_id,
            risk_score=float(-risk_score),
            is_anomaly=(prediction == -1),
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Burnout Detection

```python
# ml-service/main.py (continued)
from datetime import datetime

class WorkloadData(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_working_hours: float
    overtime_hours: float
    task_deadline_misses: int

class BurnoutAnalysis(BaseModel):
    user_id: str
    burnout_risk: str  # low, medium, high
    confidence: float
    recommendations: List[str]

@app.post("/api/ml/burnout-detection", response_model=BurnoutAnalysis)
async def detect_burnout(workload: WorkloadData):
    """Analyze burnout risk based on workload patterns"""
    try:
        score = 0
        recommendations = []
        
        # Calculate workload ratio
        workload_ratio = workload.tasks_assigned / max(workload.tasks_completed, 1)
        if workload_ratio > 2:
            score += 30
            recommendations.append("Reduce task assignments")
        
        # Check working hours
        if workload.avg_working_hours > 50:
            score += 25
            recommendations.append("Encourage work-life balance")
        
        # Check overtime
        if workload.overtime_hours > 10:
            score += 20
            recommendations.append("Monitor overtime hours")
        
        # Check deadline misses
        if workload.task_deadline_misses > 3:
            score += 25
            recommendations.append("Review task complexity and deadlines")
        
        # Determine risk level
        if score >= 70:
            risk = "high"
        elif score >= 40:
            risk = "medium"
        else:
            risk = "low"
        
        confidence = min(score / 100, 1.0)
        
        return BurnoutAnalysis(
            user_id=workload.user_id,
            burnout_risk=risk,
            confidence=confidence,
            recommendations=recommendations if recommendations else ["No immediate concerns"]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Ticket Classification

```python
# ml-service/main.py (continued)
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB

class TicketData(BaseModel):
    ticket_id: str
    title: str
    description: str

class TicketClassification(BaseModel):
    ticket_id: str
    category: str
    priority: str
    suggested_assignee: Optional[str]

@app.post("/api/ml/classify-ticket", response_model=TicketClassification)
async def classify_ticket(ticket: TicketData):
    """Classify support ticket and suggest routing"""
    try:
        text = f"{ticket.title} {ticket.description}".lower()
        
        # Simple keyword-based classification
        categories = {
            'technical': ['error', 'bug', 'crash', 'api', 'database'],
            'access': ['login', 'password', 'permission', 'access', 'authentication'],
            'feature': ['request', 'feature', 'enhancement', 'new'],
            'general': []
        }
        
        priority_keywords = {
            'high': ['urgent', 'critical', 'down', 'broken', 'not working'],
            'medium': ['issue', 'problem', 'help'],
            'low': ['question', 'how to', 'inquiry']
        }
        
        # Determine category
        category = 'general'
        for cat, keywords in categories.items():
            if any(kw in text for kw in keywords):
                category = cat
                break
        
        # Determine priority
        priority = 'low'
        for pri, keywords in priority_keywords.items():
            if any(kw in text for kw in keywords):
                priority = pri
                break
        
        # Suggest assignee based on category
        assignee_map = {
            'technical': 'tech-support',
            'access': 'admin',
            'feature': 'product-team',
            'general': 'customer-support'
        }
        
        return TicketClassification(
            ticket_id=ticket.ticket_id,
            category=category,
            priority=priority,
            suggested_assignee=assignee_map.get(category)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend Integration Patterns

### API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

// Create axios instance with auth token
const api = axios.create({
  baseURL: API_URL,
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// User API
export const userAPI = {
  getAll: () => api.get('/api/users'),
  getById: (id) => api.get(`/api/users/${id}`),
  create: (userData) => api.post('/api/users', userData),
  update: (id, userData) => api.put(`/api/users/${id}`, userData),
  delete: (id) => api.delete(`/api/users/${id}`),
};

// Task API
export const taskAPI = {
  getMyTasks: () => api.get('/api/tasks/my-tasks'),
  create: (taskData) => api.post('/api/tasks', taskData),
  updateStatus: (id, status) => api.patch(`/api/tasks/${id}/status`, { status }),
  delete: (id) => api.delete(`/api/tasks/${id}`),
};

// ML API
export const mlAPI = {
  detectRisk: (behaviorData) => 
    axios.post(`${ML_URL}/api/ml/risk-detection`, behaviorData),
  
  detectBurnout: (workloadData) => 
    axios.post(`${ML_URL}/api/ml/burnout-detection`, workloadData),
  
  classifyTicket: (ticketData) => 
    axios.post(`${ML_URL}/api/ml/classify-ticket`, ticketData),
};

export default api;
```

### Task Board Component

```javascript
// frontend/src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';

const TaskBoard = () => {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await taskAPI.getMyTasks();
      setTasks(response.data);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await taskAPI.updateStatus(taskId, newStatus);
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const tasksByStatus = {
    todo: tasks.filter(t => t.status === 'todo'),
    in_progress: tasks.filter(t => t.status === 'in_progress'),
    done: tasks.filter(t => t.status === 'done'),
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="task-board">
      {['todo', 'in_progress', 'done'].map(status => (
        <div key={status} className="task-column">
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          <div className="task-list">
            {tasksByStatus[status].map(task => (
              <div key={task._id} className="task-card">
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <span className={`priority-${task.priority}`}>
                  {task.priority}
                </span>
                <select
                  value={task.status}
                  onChange={(e) => updateTaskStatus(task._id, e.target.value)}
                >
                  <option value="todo">To Do</option>
                  <option value="in_progress">In Progress</option>
                  <option value="done">Done</option>
                </select>
              </div>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { mlAPI } from '../services/api';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch user behavior data (would come from backend in production)
      const behaviorData = {
        user_id: userId,
        login_frequency: 5.2,
        task_completion_rate: 0.75,
        avg_session_duration: 180,
        failed_logins: 1,
        data_access_count: 45
      };

      const riskResponse = await mlAPI.detectRisk(behaviorData);

      const workloadData = {
        user_id: userId,
        tasks_assigned: 15,
        tasks_completed: 12,
        avg_working_hours: 42,
        overtime_hours: 5,
        task_deadline_misses: 1
      };

      const burnoutResponse = await mlAPI.detectBurnout(workloadData);

      setAnalytics({
        risk: riskResponse.data,
        burnout: burnoutResponse.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="risk-analysis">
        <h3>Risk Analysis</h3>
        <div className={`risk-indicator ${analytics.risk.is_anomaly ? 'high' : 'low'}`}>
          <p>Risk Score: {analytics.risk.risk_score.toFixed(2)}</p>
          <p>Status: {analytics.risk.is_anomaly ? 'Anomaly Detected' : 'Normal'}</p>
          {analytics.risk.factors.length > 0 && (
            <ul>
              {analytics.risk.factors.map((factor, idx) => (
                <li key={idx}>{factor}</li>
              ))}
            </ul>
          )}
        </div>
      </div>

      <div className="burnout-analysis">
        <h3>Burnout Risk</h3>
        <div className={`burnout-indicator risk-${analytics.burnout.burnout_risk}`}>
          <p>Risk Level: {analytics.burnout.burnout_risk.toUpperCase()}</p>
          <p>Confidence: {(analytics.burnout.confidence * 100).toFixed(1)}%</p>
          <div className="recommendations">
            <h4>Recommendations:</h4>
            <ul>
              {analytics.burnout.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        </div>
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
const bcrypt = require('bcrypt');

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
    required: true
  },
  role: {
    type: String,
    enum: ['admin', 'user', 'manager'],
    default: 'user'
  },
  department: {
    type: String,
    required: true
  },
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

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

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
  description: String,
  status: {
    type: String,
    enum: ['todo', 'in_progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: Date,
  timeSpent: {
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

taskSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

module.exports = mongoose.model('Task', taskSchema);
```

## Configuration

### Environment Variables

**Backend (.env)**
```
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_management
JWT_SECRET=<generate_secure_random_string>
JWT_EXPIRATION=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**
```
MODEL_PATH=./models
LOG_LEVEL=INFO
MAX_WORKERS=4
```

**Frontend (.env)**
```
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

## Common Patterns

### Adding a New ML Model

```python
# ml-service/models/custom_model.py
from sklearn.base import BaseEstimator
import joblib
import os

class CustomPredictor(BaseEstimator):
    def __init__(self):
        self.model = None
        
    def train(self, X, y):
        """Train the model"""
        from sklearn.ensemble import RandomForestClassifier
        self.model = RandomForestClassifier(n_estimators=100)
        self.model.fit(X, y)
        
    def predict(self, X):
        """Make predictions"""
        if self.model is None:
            raise ValueError("Model not trained")
        return self.model.predict(X)
    
    def save(self, path):
        """Save model to disk"""
        joblib.dump(self.model, path)
    
    def load(self, path):
        """Load model from disk"""
        if os.path.exists(path):
            self.model = joblib.load(path)
            return True
        return False

# Add endpoint in main.py
from models.custom_model import CustomPredictor

predictor = CustomPredictor()

@app.post("/api/ml/custom-prediction")
async def custom_prediction(data: dict):
    # Implementation here
    pass
```

### Implementing Real-time Notifications

```javascript
// backend/services/notifications.js
const EventEmitter = require('events');

class NotificationService extends EventEmitter {
  constructor() {
    super();
    this.subscribers = new Map();
  }

  subscribe(userId, callback) {
    this.subscribers.set(userId, callback);
  }

  unsubscribe(userId) {
    this.subscribers.delete(userId);
  }

  notify(userId, notification) {
    const callback = this.subscribers.get(userId);
    if (callback) {
      callback(notification);
    }
    this.emit('notification', { userId, notification });
  }

  notifyTaskAssignment(task) {
    this.notify(task.assignedTo, {
      type: 'task_assigned',
      message: `New task assigned: ${task.title}`,
      taskId: task._id,
      timestamp: new Date()
    });
  }

  notifyBurnoutRisk(userId, analysis) {
    if (analysis.burnout_risk === 'high') {
      this.notify(userId, {
        type: 'burnout_alert',
        message: 'High burnout risk detected',
        recommendations: analysis.recommendations,
        timestamp: new Date()
      });
    }
  }
}

module.exports = new NotificationService();
```

## Troubleshooting

### JWT Token Issues

```javascript
// Verify token manually
const jwt = require('jsonwebtoken');

function debugToken(token) {
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    console.log('Token valid:', decoded);
    return decoded;
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      console.error('Token expired at:', error.expiredAt);
    } else if (error.name === 'JsonWebTokenError') {
      console.error('Invalid token:', error.message);
    }
    return null;
  }
}
```

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const options = {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    };

    await mongoose.connect(process.env.MONGODB_URI, options);
    console.log('MongoDB connected successfully');

    mongoose.connection.on('error', (err) => {
      console.error('MongoDB connection error:', err);
    });

    mongoose.connection.on('disconnected', () => {
      console.warn('MongoDB disconnected. Attempting to reconnect...');
    });

  } catch (error) {
    console.error('MongoDB connection failed:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Service Not Responding

```bash
# Check if service is running
curl http://localhost:8000/docs

# Test endpoint directly
curl -X POST http://localhost:8000/api/ml/risk-detection \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "test123",
    "login_frequency": 5.0,
    "task_completion_rate": 0.8,
    "avg_session_duration": 120,
    "failed_logins": 0,
    "data_access_count": 30
  }'

# Check logs
tail -f ml-service.log
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

const corsOptions = {
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));

// For development, allow all origins
if (process.env.NODE_ENV === 'development') {
  app.use(cors());
}
```

### Model Training Issues

```python
# ml-service/utils/diagnostics.py
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def diagnose_model(X, y, model):
    """Diagnose model training issues"""
    logger.info(f"Feature shape: {X.shape}")
    logger.info(f"Target shape: {y.shape}")
    logger.info(f"Feature range: min={X.min()}, max={X.max()}")
    
    # Check for NaN values
    if np.isnan(X).any():
        logger.warning("NaN values detected in features")
        
    # Check class balance
    unique, counts = np.unique(y, return_counts=True)
    logger.info(f"Class distribution: {dict(zip(unique,
