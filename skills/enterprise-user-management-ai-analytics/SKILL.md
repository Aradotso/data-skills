---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management built with React, Node.js, and FastAPI
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics for user behavior monitoring
  - implement JWT authentication with role-based access
  - create a task management system with Kanban board
  - add AI-powered ticket classification and routing
  - build user dashboards with analytics and insights
  - configure machine learning risk prediction service
  - setup MongoDB with user and task management
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System, a full-stack application that combines user administration, task management, support ticketing, and AI-powered analytics. The system uses React for frontend, Node.js for backend APIs, FastAPI for ML services, and MongoDB for data persistence.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Admin controls for adding, updating, and deleting users with role-based access
- **Task Management**: Kanban-style task board with time tracking and progress monitoring
- **Support Ticketing**: AI-powered ticket classification and intelligent routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay prediction
- **Authentication**: JWT-based secure authentication with role management
- **Dashboards**: Separate interfaces for users and administrators with real-time insights

## Installation

### Prerequisites

- Node.js (v14+)
- Python (v3.8+)
- MongoDB (local or cloud instance)

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

Create `.env` file:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRES_IN=7d
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

Create `.env` file:

```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Key Architecture Components

### Backend API Structure

The Node.js backend follows this structure:

```javascript
// backend/server.js
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
}).then(() => console.log('MongoDB connected'))
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
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true
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
    enum: ['user', 'admin', 'manager'],
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

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user || user.status !== 'active') {
      return res.status(401).json({ error: 'Invalid authentication' });
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Access denied' });
    }
    next();
  };
};

module.exports = { authenticate, authorize };
```

### Task Model and Routes

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
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
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
  timeTracked: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authenticate, authorize } = require('../middleware/auth');

// Get all tasks (user sees only their tasks)
router.get('/', authenticate, async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { assignedTo: req.user._id };
    
    const tasks = await Task.find(query)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username')
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task (admin only)
router.post('/', authenticate, authorize('admin', 'manager'), async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authenticate, async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    if (task.assignedTo.toString() !== req.user._id.toString() && req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Access denied' });
    }

    task.status = status;
    task.updatedAt = Date.now();
    await task.save();

    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update time tracked
router.patch('/:id/time', authenticate, async (req, res) => {
  try {
    const { timeTracked } = req.body;
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { $inc: { timeTracked } },
      { new: true }
    );
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### Ticket Model

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
  aiClassified: {
    type: Boolean,
    default: false
  },
  aiConfidence: Number,
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## ML Service Implementation

### FastAPI ML Service Structure

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from datetime import datetime
import os

app = FastAPI(title="Enterprise AI Analytics Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models directory
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    userId: str
    tasksCompleted: int
    tasksOverdue: int
    averageCompletionTime: float
    ticketsRaised: int
    loginFrequency: int

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    hoursWorked: float
    tasksAssigned: int
    tasksCompleted: int
    overtimeHours: float

@app.get("/")
def read_root():
    return {"status": "ML Service Running", "version": "1.0.0"}

@app.post("/api/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """
    Classify ticket category and priority using AI
    """
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple rule-based classification (replace with ML model)
        urgent_keywords = ['urgent', 'critical', 'down', 'emergency', 'asap']
        technical_keywords = ['bug', 'error', 'crash', 'api', 'database']
        billing_keywords = ['payment', 'invoice', 'billing', 'charge']
        
        category = 'general'
        priority = 'medium'
        confidence = 0.7
        
        if any(keyword in text for keyword in urgent_keywords):
            priority = 'high'
            category = 'urgent'
            confidence = 0.9
        elif any(keyword in text for keyword in technical_keywords):
            category = 'technical'
            confidence = 0.85
        elif any(keyword in text for keyword in billing_keywords):
            category = 'billing'
            confidence = 0.8
        
        return {
            "category": category,
            "priority": priority,
            "confidence": confidence,
            "aiClassified": True
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """
    Predict user risk score based on behavior patterns
    """
    try:
        # Calculate risk score (0-100)
        overdue_ratio = request.tasksOverdue / max(request.tasksCompleted, 1)
        
        risk_score = 0
        risk_factors = []
        
        if overdue_ratio > 0.3:
            risk_score += 30
            risk_factors.append("High overdue task ratio")
        
        if request.averageCompletionTime > 72:  # hours
            risk_score += 25
            risk_factors.append("Slow task completion")
        
        if request.ticketsRaised > 10:
            risk_score += 20
            risk_factors.append("High ticket volume")
        
        if request.loginFrequency < 3:  # per week
            risk_score += 25
            risk_factors.append("Low engagement")
        
        risk_level = 'low' if risk_score < 30 else 'medium' if risk_score < 60 else 'high'
        
        return {
            "userId": request.userId,
            "riskScore": min(risk_score, 100),
            "riskLevel": risk_level,
            "factors": risk_factors,
            "recommendations": generate_recommendations(risk_factors)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/detect-burnout")
async def detect_burnout(request: BurnoutAnalysisRequest):
    """
    Analyze user workload for burnout indicators
    """
    try:
        burnout_score = 0
        indicators = []
        
        # Check workload
        if request.hoursWorked > 50:
            burnout_score += 30
            indicators.append("Excessive working hours")
        
        # Check overtime
        if request.overtimeHours > 10:
            burnout_score += 25
            indicators.append("High overtime")
        
        # Check task completion rate
        completion_rate = request.tasksCompleted / max(request.tasksAssigned, 1)
        if completion_rate < 0.6:
            burnout_score += 20
            indicators.append("Low task completion rate")
        
        # Check task overload
        if request.tasksAssigned > 20:
            burnout_score += 25
            indicators.append("Task overload")
        
        status = 'healthy' if burnout_score < 30 else 'warning' if burnout_score < 60 else 'critical'
        
        return {
            "userId": request.userId,
            "burnoutScore": min(burnout_score, 100),
            "status": status,
            "indicators": indicators,
            "recommendations": generate_burnout_recommendations(status)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def generate_recommendations(risk_factors: List[str]) -> List[str]:
    recommendations = []
    if "High overdue task ratio" in risk_factors:
        recommendations.append("Review and reprioritize tasks")
    if "Slow task completion" in risk_factors:
        recommendations.append("Consider workload redistribution")
    if "High ticket volume" in risk_factors:
        recommendations.append("Provide additional support or training")
    if "Low engagement" in risk_factors:
        recommendations.append("Schedule one-on-one check-in")
    return recommendations

def generate_burnout_recommendations(status: str) -> List[str]:
    if status == 'critical':
        return [
            "Immediate workload reduction required",
            "Schedule mandatory time off",
            "Reassign non-critical tasks"
        ]
    elif status == 'warning':
        return [
            "Monitor workload closely",
            "Encourage regular breaks",
            "Review task distribution"
        ]
    return ["Maintain current work-life balance"]

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Frontend React Components

### Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUser = async () => {
    try {
      const res = await axios.get(`${API_URL}/auth/me`);
      setUser(res.data);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${API_URL}/auth/login`, { email, password });
    const { token, user } = res.data;
    localStorage.setItem('token', token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    setUser(user);
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const API_URL = process.env.REACT_APP_API_URL;

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(`${API_URL}/tasks`);
      const grouped = res.data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], 'in-progress': [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/tasks/${taskId}/status`, { status: newStatus });
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
      {['todo', 'in-progress', 'done'].map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{status.replace('-', ' ').toUpperCase()}</h3>
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
                <span className={`priority ${task.priority}`}>
                  {task.priority}
                </span>
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
// frontend/src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_URL = process.env.REACT_APP_ML_URL;
const API_URL = process.env.REACT_APP_API_URL;

const AIAnalytics = ({ userId }) => {
  const [riskData, setRiskData] = useState(null);
  const [burnoutData, setBurnoutData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      const userStats = await axios.get(`${API_URL}/analytics/user/${userId}`);
      
      const riskRes = await axios.post(`${ML_URL}/api/predict-risk`, {
        userId,
        tasksCompleted: userStats.data.tasksCompleted,
        tasksOverdue: userStats.data.tasksOverdue,
        averageCompletionTime: userStats.data.avgCompletionTime,
        ticketsRaised: userStats.data.ticketsRaised,
        loginFrequency: userStats.data.loginFrequency
      });
      setRiskData(riskRes.data);

      const burnoutRes = await axios.post(`${ML_URL}/api/detect-burnout`, {
        userId,
        hoursWorked: userStats.data.hoursWorked,
        tasksAssigned: userStats.data.tasksAssigned,
        tasksCompleted: userStats.data.tasksCompleted,
        overtimeHours: userStats.data.overtimeHours
      });
      setBurnoutData(burnoutRes.data);

      setLoading(false);
    } catch (error) {
      console.error('Error fetching analytics:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="risk-analysis">
        <h3>Risk Analysis</h3>
        <div className={`risk-score ${riskData.riskLevel}`}>
          Score: {riskData.riskScore}/100
        </div>
        <p>Level: {riskData.riskLevel.toUpperCase()}</p>
        <ul>
          {riskData.factors.map((factor, idx) => (
            <li key={idx}>{factor}</li>
          ))}
        </ul>
        <div className="recommendations">
          <h4>Recommendations:</h4>
          {riskData.recommendations.map((rec, idx) => (
            <p key={idx}>• {rec}</p>
          ))}
        </div>
      </div>

      <div className="burnout-analysis">
        <h3>Burnout Detection</h3>
        <div className={`burnout-score ${burnoutData.status}`}>
          Score: {burnoutData.burnoutScore}/100
        </div>
        <p>Status: {burnoutData.status.toUpperCase()}</p>
        <ul>
          {burnoutData.indicators.map((indicator, idx) => (
            <li key={idx}>{indicator}</li>
          ))}
        </ul>
        <div className="recommendations">
          <h4>Recommendations:</h4>
          {burnoutData.recommendations.map((rec, idx) => (
            <p key={idx}>• {rec}</p>
          ))}
        </div>
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Common Usage Patterns

### Creating a New User (Admin)

```javascript
// Example API call from admin dashboard
const createUser = async (userData) => {
  try {
    const response = await axios.post(`${API_URL}/users`, {
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role,
      department: userData.department
    });
    return response.data;
  } catch (error) {
    console.error('Error creating user:', error.response.data);
    throw error;
  }
};
```

### Raising a Support Ticket with AI Classification

```javascript
const raiseTicket = async (ticketData) => {
  try {
    // First, get AI classification
    const classification = await axios.post(`${ML_URL}/api/classify-ticket`, {
      title: ticketData.title,
      description: ticketData.description
    });

    // Create ticket with AI insights
    const ticket = await axios.post(`${API_URL}/tickets`, {
      ...ticketData,
      category: classification.data.category,
      priority: classification.data.priority,
      aiClassified: true,
      aiConfidence: classification.data.confidence
    });

    return ticket.data;
  } catch (error) {
    console.error('Error raising ticket:', error);
    throw error;
  }
};
```

## Troubleshooting

### MongoDB Connection Issues

If MongoDB connection fails:

```javascript
// Ensure MongoDB is running
// For local: mongod --dbpath /path/to/data
// For Atlas: verify MONGODB_URI in .env includes credentials

mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
  serverSelectionTimeoutMS: 5000
}).catch(err => {
  console.error('MongoDB connection error:', err);
  process.exit(1);
});
```

### CORS Issues

If frontend can't reach backend:

```javascript
// backend/server.js
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### JWT Token Expiration

Handle expired tokens gracefully:

```javascript
// frontend/src/utils/axios.js
axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### ML Service Not Responding

Check service health:

```bash
curl http://localhost:8000/
```

Verify environment variables and dependencies:

```bash
cd ml-service
pip list | grep -E "fastapi|uvicorn|pydantic"
```

### Task Time Tracking Issues

Ensure proper state management:

```javascript
const [timeTracked, setTimeTracked] = useState(0);
const [isTracking, setIsTracking] = useState(false);
const intervalRef = useRef(null);

const startTracking = () => {
  setIsTracking(true);
  intervalRef.current = setInterval(() => {
    setTimeTracked(prev => prev + 1);
  }, 1000);
};

const stopTracking = async () => {
  setIsTracking(false);
  clearInterval(intervalRef.current);
  
  // Save to backend
  await axios.patch(`${API_URL}/tasks/${taskId}/time`, {
    timeTracked
  });
};

useEffect(() => {
  return () => clearInterval(intervalRef.current);
}, []);
```

This skill provides comprehensive coverage for working with the Enterprise User Management System, including authentication, task management, AI analytics integration, and common troubleshooting scenarios.
