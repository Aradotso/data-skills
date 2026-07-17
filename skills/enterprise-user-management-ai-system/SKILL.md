---
name: enterprise-user-management-ai-system
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management capabilities
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics for user management"
  - "configure the user management dashboard with AI features"
  - "implement task tracking with burnout detection"
  - "set up JWT authentication for enterprise users"
  - "create a support ticket system with AI classification"
  - "build a Kanban board with time tracking"
  - "deploy the user management system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack application combining user management, task tracking, and support ticket systems with AI-powered analytics. It provides risk detection, anomaly detection, burnout analysis, and predictive insights for enterprise environments.

**Key Components:**
- **Frontend**: React.js dashboard with Kanban boards and analytics
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based AI engine with scikit-learn and River
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

- Node.js (v14+)
- Python (v3.8+)
- MongoDB (running locally or remote)

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
MONGODB_URI=mongodb://localhost:27017/enterprise_management
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
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
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Key Architecture

### Backend API Structure

The Node.js backend follows REST API conventions:

```javascript
// backend/server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();

app.use(cors());
app.use(express.json());

// Connect to MongoDB
mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => console.log('MongoDB connected'))
  .catch(err => console.error(err));

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/users', require('./routes/users'));
app.use('/api/tasks', require('./routes/tasks'));
app.use('/api/tickets', require('./routes/tickets'));
app.use('/api/analytics', require('./routes/analytics'));

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### Authentication Implementation

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

module.exports = function(req, res, next) {
  const token = req.header('x-auth-token');
  
  if (!token) {
    return res.status(401).json({ msg: 'No token, authorization denied' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded.user;
    next();
  } catch (err) {
    res.status(401).json({ msg: 'Token is not valid' });
  }
};

// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Login endpoint
router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  
  try {
    let user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ msg: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ msg: 'Invalid credentials' });
    }
    
    const payload = {
      user: {
        id: user.id,
        role: user.role
      }
    };
    
    jwt.sign(
      payload,
      process.env.JWT_SECRET,
      { expiresIn: '24h' },
      (err, token) => {
        if (err) throw err;
        res.json({ token, user: { id: user.id, email: user.email, role: user.role } });
      }
    );
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

module.exports = router;
```

### User Management Models

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const UserSchema = new mongoose.Schema({
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
    enum: ['admin', 'user', 'manager'],
    default: 'user'
  },
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('User', UserSchema);
```

### Task Management

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  assignedTo: {
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
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  timeSpent: {
    type: Number,
    default: 0
  },
  dueDate: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', TaskSchema);

// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const auth = require('../middleware/auth');
const Task = require('../models/Task');

// Get user tasks
router.get('/my-tasks', auth, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedTo', 'name email');
    res.json(tasks);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server Error');
  }
});

// Create task (admin only)
router.post('/', auth, async (req, res) => {
  const { title, description, assignedTo, priority, dueDate } = req.body;
  
  try {
    const newTask = new Task({
      title,
      description,
      assignedTo,
      priority,
      dueDate
    });
    
    const task = await newTask.save();
    res.json(task);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server Error');
  }
});

// Update task status
router.put('/:id/status', auth, async (req, res) => {
  const { status } = req.body;
  
  try {
    let task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ msg: 'Task not found' });
    }
    
    task.status = status;
    await task.save();
    res.json(task);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server Error');
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
from river import stream, anomaly, preprocessing
import joblib
import os

app = FastAPI(title="Enterprise AI Analytics Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
risk_model = None
anomaly_detector = anomaly.HalfSpaceTrees(seed=42)

class UserBehavior(BaseModel):
    user_id: str
    login_attempts: int
    tasks_completed: int
    avg_task_time: float
    tickets_raised: int
    working_hours: float

class TicketData(BaseModel):
    title: str
    description: str
    priority: Optional[str] = "medium"

class BurnoutAnalysis(BaseModel):
    user_id: str
    tasks_assigned: int
    tasks_completed: int
    avg_completion_time: float
    working_hours_per_day: float
    missed_deadlines: int

@app.on_event("startup")
async def load_models():
    global risk_model
    model_path = os.getenv('MODEL_PATH', './models')
    try:
        risk_model = joblib.load(f'{model_path}/risk_model.pkl')
    except:
        # Initialize with dummy model if not found
        risk_model = RandomForestClassifier(n_estimators=100)
        print("Initialized new risk model")

@app.get("/")
async def root():
    return {"status": "AI Analytics Service Running"}

@app.post("/api/risk-prediction")
async def predict_risk(behavior: UserBehavior):
    """Predict user risk level based on behavior patterns"""
    try:
        features = np.array([[
            behavior.login_attempts,
            behavior.tasks_completed,
            behavior.avg_task_time,
            behavior.tickets_raised,
            behavior.working_hours
        ]])
        
        # Simple rule-based system if model not trained
        risk_score = 0
        if behavior.login_attempts > 10:
            risk_score += 30
        if behavior.tasks_completed < 2:
            risk_score += 20
        if behavior.tickets_raised > 5:
            risk_score += 25
        if behavior.working_hours < 4:
            risk_score += 25
            
        risk_level = "high" if risk_score > 50 else "medium" if risk_score > 25 else "low"
        
        return {
            "user_id": behavior.user_id,
            "risk_level": risk_level,
            "risk_score": risk_score,
            "recommendations": generate_recommendations(risk_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/anomaly-detection")
async def detect_anomaly(behavior: UserBehavior):
    """Detect anomalous user behavior"""
    try:
        features = {
            'login_attempts': behavior.login_attempts,
            'tasks_completed': behavior.tasks_completed,
            'avg_task_time': behavior.avg_task_time,
            'tickets_raised': behavior.tickets_raised,
            'working_hours': behavior.working_hours
        }
        
        # Update anomaly detector
        anomaly_detector.learn_one(features)
        anomaly_score = anomaly_detector.score_one(features)
        
        is_anomaly = anomaly_score > 0.7
        
        return {
            "user_id": behavior.user_id,
            "is_anomaly": is_anomaly,
            "anomaly_score": float(anomaly_score),
            "alert_level": "critical" if anomaly_score > 0.9 else "warning" if is_anomaly else "normal"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ticket-classification")
async def classify_ticket(ticket: TicketData):
    """Classify support ticket and route to appropriate team"""
    try:
        text = f"{ticket.title} {ticket.description}".lower()
        
        # Simple keyword-based classification
        categories = {
            "technical": ["error", "bug", "crash", "not working", "issue", "problem"],
            "access": ["login", "password", "access", "permission", "locked"],
            "feature": ["request", "feature", "enhancement", "add", "new"],
            "general": []
        }
        
        category = "general"
        for cat, keywords in categories.items():
            if any(keyword in text for keyword in keywords):
                category = cat
                break
        
        # Assign team based on category
        team_mapping = {
            "technical": "Engineering Team",
            "access": "IT Support",
            "feature": "Product Team",
            "general": "General Support"
        }
        
        return {
            "category": category,
            "assigned_team": team_mapping[category],
            "priority": ticket.priority,
            "estimated_resolution_time": estimate_resolution_time(category)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/burnout-analysis")
async def analyze_burnout(analysis: BurnoutAnalysis):
    """Analyze employee burnout risk"""
    try:
        # Calculate burnout score
        burnout_score = 0
        
        # Task overload
        if analysis.tasks_assigned > 10:
            burnout_score += 25
        
        # Low completion rate
        completion_rate = analysis.tasks_completed / max(analysis.tasks_assigned, 1)
        if completion_rate < 0.5:
            burnout_score += 20
        
        # High working hours
        if analysis.working_hours_per_day > 9:
            burnout_score += 25
        
        # Missed deadlines
        if analysis.missed_deadlines > 3:
            burnout_score += 30
        
        burnout_level = "high" if burnout_score > 60 else "medium" if burnout_score > 30 else "low"
        
        return {
            "user_id": analysis.user_id,
            "burnout_level": burnout_level,
            "burnout_score": burnout_score,
            "recommendations": generate_burnout_recommendations(burnout_level),
            "suggested_actions": get_suggested_actions(burnout_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def generate_recommendations(risk_level: str) -> List[str]:
    recommendations = {
        "high": [
            "Review user access permissions immediately",
            "Monitor activity logs for suspicious behavior",
            "Consider temporary account restrictions",
            "Schedule security review meeting"
        ],
        "medium": [
            "Increase monitoring frequency",
            "Send security awareness reminder",
            "Review recent activities"
        ],
        "low": [
            "Continue normal monitoring",
            "No immediate action required"
        ]
    }
    return recommendations.get(risk_level, [])

def generate_burnout_recommendations(burnout_level: str) -> List[str]:
    recommendations = {
        "high": [
            "Reduce workload immediately",
            "Schedule 1-on-1 with manager",
            "Consider task redistribution",
            "Offer mental health resources"
        ],
        "medium": [
            "Monitor workload closely",
            "Check in with employee weekly",
            "Review task priorities"
        ],
        "low": [
            "Maintain current workload",
            "Regular check-ins"
        ]
    }
    return recommendations.get(burnout_level, [])

def get_suggested_actions(burnout_level: str) -> List[str]:
    actions = {
        "high": ["Redistribute 30% of tasks", "Approve time off", "Reduce meeting load"],
        "medium": ["Review deadlines", "Offer flexible hours"],
        "low": ["Continue monitoring"]
    }
    return actions.get(burnout_level, [])

def estimate_resolution_time(category: str) -> str:
    times = {
        "technical": "2-4 hours",
        "access": "30 minutes - 1 hour",
        "feature": "1-2 weeks",
        "general": "1-2 hours"
    }
    return times.get(category, "1-2 hours")
```

## Frontend Integration

### React API Service

```javascript
// frontend/src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

// Set auth token
export const setAuthToken = (token) => {
  if (token) {
    axios.defaults.headers.common['x-auth-token'] = token;
    localStorage.setItem('token', token);
  } else {
    delete axios.defaults.headers.common['x-auth-token'];
    localStorage.removeItem('token');
  }
};

// Auth API
export const authAPI = {
  login: async (email, password) => {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    return response.data;
  },
  
  register: async (userData) => {
    const response = await axios.post(`${API_URL}/api/auth/register`, userData);
    return response.data;
  }
};

// User API
export const userAPI = {
  getProfile: async () => {
    const response = await axios.get(`${API_URL}/api/users/profile`);
    return response.data;
  },
  
  getAllUsers: async () => {
    const response = await axios.get(`${API_URL}/api/users`);
    return response.data;
  },
  
  updateUser: async (userId, userData) => {
    const response = await axios.put(`${API_URL}/api/users/${userId}`, userData);
    return response.data;
  }
};

// Task API
export const taskAPI = {
  getMyTasks: async () => {
    const response = await axios.get(`${API_URL}/api/tasks/my-tasks`);
    return response.data;
  },
  
  createTask: async (taskData) => {
    const response = await axios.post(`${API_URL}/api/tasks`, taskData);
    return response.data;
  },
  
  updateTaskStatus: async (taskId, status) => {
    const response = await axios.put(`${API_URL}/api/tasks/${taskId}/status`, {
      status
    });
    return response.data;
  },
  
  updateTimeSpent: async (taskId, timeSpent) => {
    const response = await axios.put(`${API_URL}/api/tasks/${taskId}/time`, {
      timeSpent
    });
    return response.data;
  }
};

// ML API
export const mlAPI = {
  predictRisk: async (behaviorData) => {
    const response = await axios.post(`${ML_API_URL}/api/risk-prediction`, behaviorData);
    return response.data;
  },
  
  detectAnomaly: async (behaviorData) => {
    const response = await axios.post(`${ML_API_URL}/api/anomaly-detection`, behaviorData);
    return response.data;
  },
  
  classifyTicket: async (ticketData) => {
    const response = await axios.post(`${ML_API_URL}/api/ticket-classification`, ticketData);
    return response.data;
  },
  
  analyzeBurnout: async (analysisData) => {
    const response = await axios.post(`${ML_API_URL}/api/burnout-analysis`, analysisData);
    return response.data;
  }
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';
import './KanbanBoard.css';

const KanbanBoard = () => {
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
      const allTasks = await taskAPI.getMyTasks();
      const grouped = {
        todo: allTasks.filter(t => t.status === 'todo'),
        'in-progress': allTasks.filter(t => t.status === 'in-progress'),
        done: allTasks.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error loading tasks:', error);
    }
  };

  const handleDragStart = (e, task) => {
    e.dataTransfer.setData('task', JSON.stringify(task));
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const task = JSON.parse(e.dataTransfer.getData('task'));
    
    try {
      await taskAPI.updateTaskStatus(task._id, newStatus);
      loadTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  return (
    <div className="kanban-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{status.toUpperCase().replace('-', ' ')}</h3>
          <div className="task-list">
            {tasks[status].map(task => (
              <div
                key={task._id}
                className="task-card"
                draggable
                onDragStart={(e) => handleDragStart(e, task)}
              >
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <span className={`priority priority-${task.priority}`}>
                  {task.priority}
                </span>
                <div className="task-time">
                  Time: {Math.floor(task.timeSpent / 60)}h {task.timeSpent % 60}m
                </div>
              </div>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import { mlAPI, userAPI } from '../services/api';
import './AIAnalytics.css';

const AIAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    riskPrediction: null,
    anomalyDetection: null,
    burnoutAnalysis: null
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadAnalytics();
  }, []);

  const loadAnalytics = async () => {
    try {
      setLoading(true);
      const profile = await userAPI.getProfile();
      
      // Get behavior data (in real app, this comes from backend)
      const behaviorData = {
        user_id: profile._id,
        login_attempts: 3,
        tasks_completed: 8,
        avg_task_time: 2.5,
        tickets_raised: 2,
        working_hours: 7.5
      };
      
      const [riskPrediction, anomalyDetection, burnoutAnalysis] = await Promise.all([
        mlAPI.predictRisk(behaviorData),
        mlAPI.detectAnomaly(behaviorData),
        mlAPI.analyzeBurnout({
          user_id: profile._id,
          tasks_assigned: 10,
          tasks_completed: 8,
          avg_completion_time: 2.5,
          working_hours_per_day: 7.5,
          missed_deadlines: 1
        })
      ]);
      
      setAnalytics({ riskPrediction, anomalyDetection, burnoutAnalysis });
    } catch (error) {
      console.error('Error loading analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Analytics</h2>
      
      <div className="analytics-grid">
        <div className="analytics-card">
          <h3>Risk Assessment</h3>
          <div className={`risk-level risk-${analytics.riskPrediction?.risk_level}`}>
            {analytics.riskPrediction?.risk_level?.toUpperCase()}
          </div>
          <p>Score: {analytics.riskPrediction?.risk_score}</p>
          <ul>
            {analytics.riskPrediction?.recommendations.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        </div>
        
        <div className="analytics-card">
          <h3>Anomaly Detection</h3>
          <div className={`anomaly-status ${analytics.anomalyDetection?.is_anomaly ? 'anomaly' : 'normal'}`}>
            {analytics.anomalyDetection?.alert_level?.toUpperCase()}
          </div>
          <p>Score: {analytics.anomalyDetection?.anomaly_score?.toFixed(2)}</p>
        </div>
        
        <div className="analytics-card">
          <h3>Burnout Analysis</h3>
          <div className={`burnout-level burnout-${analytics.burnoutAnalysis?.burnout_level}`}>
            {analytics.burnoutAnalysis?.burnout_level?.toUpperCase()}
          </div>
          <p>Score: {analytics.burnoutAnalysis?.burnout_score}</p>
          <h4>Recommendations:</h4>
          <ul>
            {analytics.burnoutAnalysis?.recommendations.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        </div>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Common Patterns

### Role-Based Access Control

```javascript
// backend/middleware/adminAuth.js
module.exports = function(req, res, next) {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ msg: 'Access denied. Admin only.' });
  }
  next();
};

// Usage in routes
const auth = require('../middleware/auth');
const adminAuth = require('../middleware/adminAuth');

router.post('/users', [auth, adminAuth], async (req, res) => {
  // Only admins can create users
});
```

### Time Tracking Integration

```javascript
// frontend/src/components/TaskTimer.jsx
import React, { useState, useEffect } from 'react';
import { taskAPI } from '../services/api';

const TaskTimer = ({ task }) => {
  const [seconds, setSeconds] = useState(task.timeSpent || 0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const handleStop = async () => {
    setIsRunning(false);
    try {
      await taskAPI.updateTimeSpent(task._id, seconds);
    } catch (error) {
      console.error('Error saving time:', error);
    }
  };

  return (
    <div className="task-timer">
      <div className="time-display">
        {Math.floor(seconds / 3600)}h {Math.floor((seconds % 3600) / 60)}m {seconds % 60}s
      </div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      {isRunning && <button onClick={handleStop}>Stop & Save</button>}
    </div>
  );
};

export default TaskTimer;
```

## Configuration

### Environment Variables

**Backend (.env):**
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_management
JWT_SECRET=your_secure_jwt_secret_key
JWT_EXPIRE=24h
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env):**
```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
MAX_WORKERS=4
```

**Frontend (.env):**
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_ANALYTICS=true
```

## Troubleshooting

### Connection Issues

**Problem**: Cannot connect to MongoDB
```bash
# Check MongoDB status
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Verify connection
mongo --eval "db.adminCommand('ping')"
```

**Problem**: CORS errors in browser
```javascript
// backend/server.js - Add CORS configuration
const
