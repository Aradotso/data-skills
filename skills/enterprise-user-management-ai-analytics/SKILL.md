---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and task optimization
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics for user management
  - implement task tracking with AI insights
  - create user management dashboard with ML
  - add anomaly detection to user system
  - build admin panel with AI analytics
  - configure JWT authentication for enterprise app
  - implement burnout detection for users
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack enterprise user management system built with React, Node.js, FastAPI, and MongoDB. It provides comprehensive user, task, and ticket management with AI-powered features including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## What It Does

- **User Management**: Role-based access control, user CRUD operations, authentication with JWT
- **Task Management**: Kanban board (To Do/In Progress/Done), time tracking, assignment workflows
- **Support System**: Ticket creation, tracking, and AI-based classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, predictive insights
- **Admin Dashboard**: Organization analytics, audit logs, user monitoring

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB running locally or remote connection
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
EOF

# Start backend server
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
EOF

# Start FastAPI ML service
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

# Start React app
npm start
```

## Architecture Overview

```
Frontend (React) → Backend (Node.js) → Database (MongoDB)
                ↓
          ML Service (FastAPI) → AI Models
```

## Backend API Reference

### Authentication Endpoints

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Register new user
router.post('/register', async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const userExists = await User.findOne({ email });
    if (userExists) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const user = await User.create({ name, email, password, role });
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });
    
    res.status(201).json({ success: true, token, user });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Login user
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email }).select('+password');
    if (!user || !(await user.matchPassword(password))) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE
    });
    
    res.json({ success: true, token, user });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### User Management

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');
const User = require('../models/User');

// Get all users (admin only)
router.get('/', protect, authorize('admin'), async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ success: true, count: users.length, data: users });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update user
router.put('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    );
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Delete user
router.delete('/:id', protect, authorize('admin'), async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json({ success: true, message: 'User deleted' });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

### Task Management

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const { protect } = require('../middleware/auth');
const Task = require('../models/Task');

// Get user tasks
router.get('/', protect, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('assignedBy', 'name email')
      .sort('-createdAt');
    
    res.json({ success: true, count: tasks.length, data: tasks });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Create task
router.post('/', protect, async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      assignedBy: req.user.id
    });
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task status
router.patch('/:id/status', protect, async (req, res) => {
  try {
    const { status } = req.body; // 'todo', 'inProgress', 'done'
    
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Track time on task
router.post('/:id/time', protect, async (req, res) => {
  try {
    const { timeSpent } = req.body; // in minutes
    
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { $inc: { timeTracked: timeSpent } },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json({ success: true, data: task });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

module.exports = router;
```

## ML Service API

### AI Analytics Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import IsolationForest
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Models
class UserBehavior(BaseModel):
    user_id: str
    login_times: List[float]
    task_completion_rate: float
    avg_task_time: float
    failed_logins: int
    active_hours: List[int]

class TaskData(BaseModel):
    task_id: str
    assigned_to: str
    estimated_hours: float
    complexity: int
    dependencies: int

class TicketData(BaseModel):
    title: str
    description: str
    priority: Optional[str] = None

# Anomaly Detection
@app.post("/ai/detect-anomaly")
async def detect_anomaly(data: UserBehavior):
    """Detect anomalous user behavior patterns"""
    try:
        # Feature engineering
        features = np.array([[
            data.task_completion_rate,
            data.avg_task_time,
            data.failed_logins,
            len(data.active_hours),
            np.std(data.login_times) if len(data.login_times) > 1 else 0
        ]])
        
        # Load or create model
        model_path = os.path.join(os.getenv('MODEL_PATH', './models'), 'anomaly_detector.pkl')
        if os.path.exists(model_path):
            model = joblib.load(model_path)
        else:
            model = IsolationForest(contamination=0.1, random_state=42)
            # In production, train with historical data
            model.fit(features)
            joblib.dump(model, model_path)
        
        prediction = model.predict(features)[0]
        anomaly_score = model.score_samples(features)[0]
        
        return {
            "is_anomaly": bool(prediction == -1),
            "anomaly_score": float(anomaly_score),
            "risk_level": "high" if prediction == -1 else "low",
            "user_id": data.user_id
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Burnout Detection
@app.post("/ai/detect-burnout")
async def detect_burnout(data: UserBehavior):
    """Predict user burnout based on workload patterns"""
    try:
        # Calculate burnout indicators
        avg_hours_per_day = len(data.active_hours) / 7 if data.active_hours else 0
        task_stress = (1 - data.task_completion_rate) * 100
        overtime_score = max(0, avg_hours_per_day - 8) * 10
        
        burnout_score = (
            task_stress * 0.4 +
            overtime_score * 0.4 +
            data.failed_logins * 5 * 0.2
        )
        
        burnout_risk = "high" if burnout_score > 60 else "medium" if burnout_score > 30 else "low"
        
        return {
            "burnout_score": round(burnout_score, 2),
            "risk_level": burnout_risk,
            "avg_hours_per_day": round(avg_hours_per_day, 2),
            "task_completion_rate": data.task_completion_rate,
            "recommendations": generate_burnout_recommendations(burnout_risk)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def generate_burnout_recommendations(risk_level: str) -> List[str]:
    recommendations = {
        "high": [
            "Redistribute workload immediately",
            "Schedule mandatory break period",
            "Review task assignments with manager",
            "Consider reducing overtime hours"
        ],
        "medium": [
            "Monitor workload trends closely",
            "Encourage regular breaks",
            "Review task priorities"
        ],
        "low": [
            "Maintain current work balance",
            "Continue regular check-ins"
        ]
    }
    return recommendations.get(risk_level, [])

# Risk Prediction
@app.post("/ai/predict-risk")
async def predict_risk(data: UserBehavior):
    """Predict security and performance risks"""
    try:
        risk_factors = {
            "security": 0,
            "performance": 0,
            "compliance": 0
        }
        
        # Security risk
        if data.failed_logins > 3:
            risk_factors["security"] += 40
        if len(set(data.active_hours)) > 18:  # Working at unusual hours
            risk_factors["security"] += 20
        
        # Performance risk
        if data.task_completion_rate < 0.7:
            risk_factors["performance"] += 50
        if data.avg_task_time > 8:
            risk_factors["performance"] += 30
        
        overall_risk = sum(risk_factors.values()) / len(risk_factors)
        
        return {
            "overall_risk_score": round(overall_risk, 2),
            "risk_breakdown": risk_factors,
            "risk_level": "critical" if overall_risk > 70 else "high" if overall_risk > 40 else "medium" if overall_risk > 20 else "low",
            "user_id": data.user_id
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Ticket Classification
@app.post("/ai/classify-ticket")
async def classify_ticket(ticket: TicketData):
    """Classify and route support tickets using AI"""
    try:
        text = f"{ticket.title} {ticket.description}".lower()
        
        # Simple keyword-based classification (can be enhanced with NLP)
        categories = {
            "technical": ["bug", "error", "crash", "not working", "broken"],
            "account": ["password", "login", "access", "permission"],
            "feature": ["request", "enhancement", "improvement", "suggestion"],
            "urgent": ["critical", "urgent", "emergency", "down"]
        }
        
        classification = {"category": "general", "priority": "medium", "department": "support"}
        
        for category, keywords in categories.items():
            if any(keyword in text for keyword in keywords):
                classification["category"] = category
                if category == "urgent":
                    classification["priority"] = "high"
                break
        
        # Determine department routing
        if classification["category"] == "technical":
            classification["department"] = "engineering"
        elif classification["category"] == "account":
            classification["department"] = "admin"
        
        return classification
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Project Delay Prediction
@app.post("/ai/predict-delay")
async def predict_delay(task: TaskData):
    """Predict if a task/project will be delayed"""
    try:
        # Calculate delay probability based on task characteristics
        complexity_factor = task.complexity * 0.15
        dependency_factor = task.dependencies * 0.1
        
        delay_probability = min(100, (complexity_factor + dependency_factor) * 20)
        
        will_delay = delay_probability > 50
        
        return {
            "will_delay": will_delay,
            "delay_probability": round(delay_probability, 2),
            "estimated_hours": task.estimated_hours,
            "risk_factors": {
                "complexity": task.complexity,
                "dependencies": task.dependencies
            },
            "recommendations": [
                "Add buffer time to estimate" if will_delay else "Timeline appears reasonable",
                "Break down into smaller tasks" if task.complexity > 7 else "Task complexity manageable",
                "Review dependencies" if task.dependencies > 3 else "Dependencies under control"
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Frontend Integration

### React Authentication Hook

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export const useAuth = () => useContext(AuthContext);

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      loadUser();
    } else {
      setLoading(false);
    }
  }, []);

  const loadUser = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(res.data.data);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', res.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${res.data.token}`;
    setUser(res.data.user);
    return res.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, loading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`);
      const categorized = {
        todo: res.data.data.filter(t => t.status === 'todo'),
        inProgress: res.data.data.filter(t => t.status === 'inProgress'),
        done: res.data.data.filter(t => t.status === 'done')
      };
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus }
      );
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

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{status === 'todo' ? 'To Do' : status === 'inProgress' ? 'In Progress' : 'Done'}</h3>
          <div className="task-list">
            {tasks[status].map(task => (
              <div
                key={task._id}
                className="task-card"
                draggable
                onDragStart={(e) => handleDragStart(e, task._id)}
              >
                <h4>{task.title}</h4>
                <p>{task.description}</p>
                <div className="task-meta">
                  <span>Est: {task.estimatedHours}h</span>
                  <span className={`priority-${task.priority}`}>
                    {task.priority}
                  </span>
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
// frontend/src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    anomaly: null,
    burnout: null,
    risk: null
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Get user behavior data
      const behaviorRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/users/${userId}/behavior`
      );
      
      const behaviorData = behaviorRes.data.data;

      // Call ML service for analytics
      const [anomalyRes, burnoutRes, riskRes] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_URL}/ai/detect-anomaly`, behaviorData),
        axios.post(`${process.env.REACT_APP_ML_URL}/ai/detect-burnout`, behaviorData),
        axios.post(`${process.env.REACT_APP_ML_URL}/ai/predict-risk`, behaviorData)
      ]);

      setAnalytics({
        anomaly: anomalyRes.data,
        burnout: burnoutRes.data,
        risk: riskRes.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI-Powered Analytics</h2>
      
      {/* Anomaly Detection */}
      <div className={`analytics-card ${analytics.anomaly?.is_anomaly ? 'alert' : ''}`}>
        <h3>Anomaly Detection</h3>
        <p>Status: {analytics.anomaly?.is_anomaly ? '⚠️ Anomaly Detected' : '✓ Normal'}</p>
        <p>Risk Level: {analytics.anomaly?.risk_level}</p>
        <p>Score: {analytics.anomaly?.anomaly_score?.toFixed(2)}</p>
      </div>

      {/* Burnout Analysis */}
      <div className={`analytics-card risk-${analytics.burnout?.risk_level}`}>
        <h3>Burnout Analysis</h3>
        <p>Burnout Score: {analytics.burnout?.burnout_score}/100</p>
        <p>Risk Level: {analytics.burnout?.risk_level}</p>
        <p>Avg Hours/Day: {analytics.burnout?.avg_hours_per_day}</p>
        <div className="recommendations">
          <h4>Recommendations:</h4>
          <ul>
            {analytics.burnout?.recommendations?.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      </div>

      {/* Risk Prediction */}
      <div className="analytics-card">
        <h3>Risk Assessment</h3>
        <p>Overall Risk: {analytics.risk?.overall_risk_score}/100</p>
        <p>Level: {analytics.risk?.risk_level}</p>
        <div className="risk-breakdown">
          <h4>Risk Breakdown:</h4>
          <ul>
            <li>Security: {analytics.risk?.risk_breakdown?.security}</li>
            <li>Performance: {analytics.risk?.risk_breakdown?.performance}</li>
            <li>Compliance: {analytics.risk?.risk_breakdown?.compliance}</li>
          </ul>
        </div>
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Database Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please add a name']
  },
  email: {
    type: String,
    required: [true, 'Please add an email'],
    unique: true,
    match: [/^\w+([\.-]?\w+)*@\w+([\.-]?\w+)*(\.\w{2,3})+$/, 'Please add a valid email']
  },
  password: {
    type: String,
    required: [true, 'Please add a password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  position: String,
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

// Encrypt password
UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) {
    next();
  }
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

// Match password
UserSchema.methods.matchPassword = async function(enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: [true, 'Please add a task title']
  },
  description: String,
  status: {
    type: String,
    enum: ['todo', 'inProgress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical'],
    default: 'medium'
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  estimatedHours: Number,
  timeTracked: {
    type: Number,
    default: 0
  },
  complexity: {
    type: Number,
    min: 1,
    max: 10
  },
  dependencies: {
    type: Number,
    default: 0
  },
  dueDate: Date,
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: Date
});

module.exports = mongoose.model('Task', TaskSchema);
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
NODE_ENV=development
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**
```bash
MODEL_PATH=./models
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
LOG_LEVEL=info
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

## Common Patterns

### Protected Routes with Role-Based Access

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization?.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ message: 'Not authorized' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id);
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Not authorized' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: `User role ${req.user.role} is not authorized` 
      });
    }
    next();
  };
};
```

### Time Tracking Component

```javascript
// frontend/src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [time, setTime] = useState(0);
  const [isRunning,
