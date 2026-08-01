---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management with task tracking"
  - "add AI anomaly detection to user system"
  - "build admin dashboard with AI insights"
  - "integrate ML-based ticket classification"
  - "setup user management with burnout detection"
  - "deploy enterprise management system with AI"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack enterprise user management platform that combines user administration, task tracking, and support ticketing with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Secure authentication (JWT), role-based access control, user CRUD operations
- **Task Management**: Kanban board (To Do → In Progress → Done), time tracking, task assignment
- **Support Ticketing**: Ticket creation, tracking, and AI-based classification/routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, activity monitoring

**Stack**: React.js frontend, Node.js/Express backend, FastAPI ML service, MongoDB database, scikit-learn/River for ML

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB running locally or connection string

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
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
EOF

# Start backend
npm start
# Runs at http://localhost:5000
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

# Start ML service
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
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
# Runs at http://localhost:3000
```

## Backend API Reference

### Authentication Endpoints

```javascript
// backend/routes/auth.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const User = require('../models/User');

// Register user
router.post('/register', async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({
      username,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    res.status(201).json({ message: 'User created successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    
    if (!user || !(await bcrypt.compare(password, user.password))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    
    res.json({ token, user: { id: user._id, username: user.username, role: user.role } });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### User Management Endpoints

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const { authMiddleware, adminMiddleware } = require('../middleware/auth');
const User = require('../models/User');

// Get all users (admin only)
router.get('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update user
router.put('/:id', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const { username, email, role, status } = req.body;
    const user = await User.findByIdAndUpdate(
      req.params.id,
      { username, email, role, status },
      { new: true }
    ).select('-password');
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Delete user
router.delete('/:id', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    await User.findByIdAndDelete(req.params.id);
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Task Management Endpoints

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const { authMiddleware } = require('../middleware/auth');
const Task = require('../models/Task');

// Get user tasks
router.get('/my-tasks', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.userId });
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    const task = new Task({
      title,
      description,
      assignedTo,
      priority: priority || 'medium',
      status: 'todo',
      dueDate,
      createdBy: req.user.userId
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const { status } = req.body; // 'todo', 'in-progress', 'done'
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Track time
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const { duration } = req.body; // duration in seconds
    const task = await Task.findById(req.params.id);
    
    task.timeSpent = (task.timeSpent || 0) + duration;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Support Ticket Endpoints

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const axios = require('axios');
const { authMiddleware } = require('../middleware/auth');
const Ticket = require('../models/Ticket');

// Create ticket with AI classification
router.post('/', authMiddleware, async (req, res) => {
  try {
    const { subject, description, priority } = req.body;
    
    // Get AI classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/classify-ticket`,
      { subject, description }
    );
    
    const ticket = new Ticket({
      subject,
      description,
      priority: priority || 'medium',
      status: 'open',
      category: mlResponse.data.category,
      suggestedAssignee: mlResponse.data.suggestedAssignee,
      createdBy: req.user.userId
    });
    
    await ticket.save();
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get user tickets
router.get('/my-tickets', authMiddleware, async (req, res) => {
  try {
    const tickets = await Ticket.find({ createdBy: req.user.userId });
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

## ML Service API Reference

### Risk Prediction

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os

app = FastAPI()

class UserBehavior(BaseModel):
    login_frequency: float
    task_completion_rate: float
    avg_task_duration: float
    failed_login_attempts: int
    unusual_hours_access: int

class RiskPredictionResponse(BaseModel):
    risk_score: float
    risk_level: str
    factors: list

@app.post("/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(data: UserBehavior):
    try:
        # Load or create model
        model_path = os.getenv('MODEL_PATH', './models') + '/risk_model.pkl'
        if os.path.exists(model_path):
            model = joblib.load(model_path)
        else:
            # Initialize with default model
            model = RandomForestClassifier(n_estimators=100)
        
        # Prepare features
        features = np.array([[
            data.login_frequency,
            data.task_completion_rate,
            data.avg_task_duration,
            data.failed_login_attempts,
            data.unusual_hours_access
        ]])
        
        # Predict risk
        risk_score = model.predict_proba(features)[0][1] if hasattr(model, 'predict_proba') else 0.5
        
        # Determine risk level
        if risk_score > 0.7:
            risk_level = "high"
        elif risk_score > 0.4:
            risk_level = "medium"
        else:
            risk_level = "low"
        
        # Identify risk factors
        factors = []
        if data.failed_login_attempts > 3:
            factors.append("High failed login attempts")
        if data.task_completion_rate < 0.5:
            factors.append("Low task completion rate")
        if data.unusual_hours_access > 5:
            factors.append("Unusual access hours")
        
        return RiskPredictionResponse(
            risk_score=float(risk_score),
            risk_level=risk_level,
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Anomaly Detection

```python
# ml-service/anomaly_detection.py
from fastapi import APIRouter
from pydantic import BaseModel
from sklearn.ensemble import IsolationForest
import numpy as np

router = APIRouter()

class ActivityData(BaseModel):
    user_id: str
    activities: list[dict]  # [{timestamp, action, duration}]

class AnomalyResponse(BaseModel):
    is_anomaly: bool
    anomaly_score: float
    suspicious_activities: list

@router.post("/detect-anomaly", response_model=AnomalyResponse)
async def detect_anomaly(data: ActivityData):
    # Extract features from activities
    features = []
    for activity in data.activities:
        features.append([
            activity.get('hour', 0),
            activity.get('duration', 0),
            activity.get('action_type', 0)
        ])
    
    if len(features) == 0:
        return AnomalyResponse(
            is_anomaly=False,
            anomaly_score=0.0,
            suspicious_activities=[]
        )
    
    # Train isolation forest
    clf = IsolationForest(contamination=0.1, random_state=42)
    predictions = clf.fit_predict(features)
    scores = clf.score_samples(features)
    
    # Find anomalies
    suspicious = []
    for i, (pred, score) in enumerate(zip(predictions, scores)):
        if pred == -1:  # Anomaly detected
            suspicious.append({
                'timestamp': data.activities[i].get('timestamp'),
                'action': data.activities[i].get('action'),
                'score': float(score)
            })
    
    is_anomaly = len(suspicious) > 0
    avg_score = float(np.mean(scores))
    
    return AnomalyResponse(
        is_anomaly=is_anomaly,
        anomaly_score=avg_score,
        suspicious_activities=suspicious
    )
```

### Burnout Detection

```python
# ml-service/burnout_detection.py
from fastapi import APIRouter
from pydantic import BaseModel

router = APIRouter()

class WorkloadData(BaseModel):
    user_id: str
    hours_worked: float
    tasks_completed: int
    tasks_pending: int
    avg_task_duration: float
    overtime_hours: float

class BurnoutResponse(BaseModel):
    burnout_risk: str
    score: float
    recommendations: list

@router.post("/detect-burnout", response_model=BurnoutResponse)
async def detect_burnout(data: WorkloadData):
    # Calculate burnout score
    score = 0.0
    
    # Hours worked factor
    if data.hours_worked > 50:
        score += 0.3
    elif data.hours_worked > 40:
        score += 0.15
    
    # Task load factor
    task_ratio = data.tasks_pending / max(data.tasks_completed, 1)
    if task_ratio > 2:
        score += 0.25
    elif task_ratio > 1:
        score += 0.15
    
    # Overtime factor
    if data.overtime_hours > 10:
        score += 0.3
    elif data.overtime_hours > 5:
        score += 0.15
    
    # Task duration (indicator of complexity/stress)
    if data.avg_task_duration > 8:
        score += 0.15
    
    # Determine risk level
    if score > 0.6:
        risk = "high"
    elif score > 0.3:
        risk = "medium"
    else:
        risk = "low"
    
    # Generate recommendations
    recommendations = []
    if data.hours_worked > 45:
        recommendations.append("Reduce working hours")
    if task_ratio > 1.5:
        recommendations.append("Delegate or defer tasks")
    if data.overtime_hours > 5:
        recommendations.append("Minimize overtime")
    if data.avg_task_duration > 6:
        recommendations.append("Break down complex tasks")
    
    return BurnoutResponse(
        burnout_risk=risk,
        score=score,
        recommendations=recommendations
    )
```

### Ticket Classification

```python
# ml-service/ticket_classifier.py
from fastapi import APIRouter
from pydantic import BaseModel
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import re

router = APIRouter()

class TicketData(BaseModel):
    subject: str
    description: str

class ClassificationResponse(BaseModel):
    category: str
    confidence: float
    suggestedAssignee: str

# Initialize simple classifier (in production, load pre-trained model)
categories = {
    'technical': ['bug', 'error', 'crash', 'not working', 'issue'],
    'access': ['login', 'password', 'permission', 'access', 'authentication'],
    'feature': ['request', 'enhancement', 'new feature', 'improvement'],
    'billing': ['payment', 'invoice', 'subscription', 'charge', 'refund']
}

@router.post("/classify-ticket", response_model=ClassificationResponse)
async def classify_ticket(data: TicketData):
    text = f"{data.subject} {data.description}".lower()
    
    # Simple keyword-based classification
    scores = {cat: 0 for cat in categories}
    for category, keywords in categories.items():
        for keyword in keywords:
            if keyword in text:
                scores[category] += 1
    
    # Get category with highest score
    category = max(scores, key=scores.get)
    total_matches = sum(scores.values())
    confidence = scores[category] / max(total_matches, 1)
    
    # Suggest assignee based on category
    assignee_map = {
        'technical': 'tech-support',
        'access': 'admin',
        'feature': 'product-team',
        'billing': 'finance'
    }
    
    return ClassificationResponse(
        category=category,
        confidence=confidence,
        suggestedAssignee=assignee_map.get(category, 'general-support')
    )
```

## Frontend Usage Patterns

### Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      // Verify token and load user
      loadUser();
    } else {
      setLoading(false);
    }
  }, []);

  const loadUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    
    localStorage.setItem('token', response.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
    setUser(response.data.user);
    
    return response.data;
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

### Kanban Task Board

```javascript
// frontend/src/components/TaskBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks/my-tasks`);
      const grouped = response.data.reduce((acc, task) => {
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
      await axios.patch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const renderColumn = (status, title) => (
    <div className="task-column">
      <h3>{title}</h3>
      {tasks[status].map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <div className="task-actions">
            {status !== 'todo' && (
              <button onClick={() => updateTaskStatus(task._id, 
                status === 'in-progress' ? 'todo' : 'in-progress')}>
                ← Back
              </button>
            )}
            {status !== 'done' && (
              <button onClick={() => updateTaskStatus(task._id, 
                status === 'todo' ? 'in-progress' : 'done')}>
                Next →
              </button>
            )}
          </div>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do')}
      {renderColumn('in-progress', 'In Progress')}
      {renderColumn('done', 'Done')}
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// frontend/src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      setLoading(true);
      
      // Get user behavior data
      const userResponse = await axios.get(
        `${process.env.REACT_APP_API_URL}/analytics/user-behavior/${userId}`
      );
      
      // Get AI predictions
      const [riskResponse, burnoutResponse] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_API_URL}/predict-risk`, 
          userResponse.data),
        axios.post(`${process.env.REACT_APP_ML_API_URL}/detect-burnout`, 
          userResponse.data)
      ]);
      
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
  if (!analytics) return <div>No analytics available</div>;

  return (
    <div className="ai-analytics">
      <div className="risk-section">
        <h3>Risk Assessment</h3>
        <div className={`risk-badge ${analytics.risk.risk_level}`}>
          {analytics.risk.risk_level.toUpperCase()}
        </div>
        <p>Risk Score: {(analytics.risk.risk_score * 100).toFixed(1)}%</p>
        <ul>
          {analytics.risk.factors.map((factor, idx) => (
            <li key={idx}>{factor}</li>
          ))}
        </ul>
      </div>
      
      <div className="burnout-section">
        <h3>Burnout Detection</h3>
        <div className={`burnout-badge ${analytics.burnout.burnout_risk}`}>
          {analytics.burnout.burnout_risk.toUpperCase()} RISK
        </div>
        <p>Burnout Score: {(analytics.burnout.score * 100).toFixed(1)}%</p>
        <h4>Recommendations:</h4>
        <ul>
          {analytics.burnout.recommendations.map((rec, idx) => (
            <li key={idx}>{rec}</li>
          ))}
        </ul>
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

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user', 'manager'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive', 'suspended'], default: 'active' },
  createdAt: { type: Date, default: Date.now },
  lastLogin: { type: Date },
  failedLoginAttempts: { type: Number, default: 0 }
});

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  dueDate: { type: Date },
  timeSpent: { type: Number, default: 0 }, // in seconds
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['open', 'in-progress', 'resolved', 'closed'], default: 'open' },
  priority: { type: String, enum: ['low', 'medium', 'high', 'urgent'], default: 'medium' },
  category: { type: String }, // AI-classified
  suggestedAssignee: { type: String }, // AI-suggested
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Configuration

### Backend Environment Variables

```bash
# .env (backend)
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-mgmt
JWT_SECRET=your-secret-key-here
JWT_EXPIRE=24h
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
```

### ML Service Environment Variables

```bash
# .env (ml-service)
MODEL_PATH=./models
LOG_LEVEL=INFO
MAX_WORKERS=4
CACHE_ENABLED=true
```

### Frontend Environment Variables

```bash
# .env (frontend)
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Common Patterns

### Middleware for Authentication

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return res.status(401).json({ error: 'No token provided' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Audit Logging

```javascript
// backend/middleware/auditLog.js
const AuditLog = require('../models/AuditLog');

const auditMiddleware = (action) => {
  return async (req, res, next) => {
    const originalJson = res.json;
    
    res.json = function(data) {
      // Log after successful response
      AuditLog.create({
        userId: req.user?.userId,
        action,
        resource: req.originalUrl,
        method: req.method,
        timestamp: new Date(),
        ipAddress: req.ip,
        userAgent: req.get('user-agent')
      }).catch(err => console.error('Audit log error:', err));
      
      originalJson.call(this, data);
    };
    
    next();
  };
};

module.exports = auditMiddleware;
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
