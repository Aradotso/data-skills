---
name: enterprise-user-management-ai-analytics
description: AI-powered enterprise user management system with task tracking, ticket routing, and predictive analytics for organizational efficiency
triggers:
  - "build a user management system with AI analytics"
  - "create an enterprise task and ticket management platform"
  - "implement AI-based risk detection and burnout analysis"
  - "set up a kanban board with time tracking"
  - "develop role-based access control with JWT authentication"
  - "integrate machine learning for ticket classification"
  - "build an admin dashboard with user analytics"
  - "create a full-stack user management app with AI insights"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build and work with an enterprise-grade user management system that combines traditional CRUD operations with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and intelligent ticket routing.

## What This Project Does

The Enterprise User Management System is a full-stack application featuring:

- **User Management**: Role-based access control with admin and user roles
- **Task Management**: Kanban-style task boards with time tracking
- **Ticket System**: Support ticket creation and AI-based routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout detection, and project delay prediction
- **Authentication**: JWT-based secure authentication
- **Real-time Insights**: Dashboard with organizational analytics

**Architecture**: React frontend + Node.js/Express backend + FastAPI ML service + MongoDB database

## Installation

### Prerequisites

```bash
# Required software
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
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
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=24h
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=info
EOF

# Start ML service
uvicorn main:app --reload --port 8000
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
```

## Key API Endpoints

### Authentication APIs

```javascript
// POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "securePass123",
  "role": "user"
}

// POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePass123"
}

// Response
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "507f1f77bcf86cd799439011",
    "username": "john.doe",
    "role": "user"
  }
}
```

### User Management APIs

```javascript
// GET /api/users - Get all users (Admin only)
// GET /api/users/:id - Get user by ID
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user (Admin only)

// Example: Update user
const updateUser = async (userId, updates) => {
  const response = await fetch(`${API_URL}/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

### Task Management APIs

```javascript
// POST /api/tasks - Create task
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth system",
  "assignedTo": "507f1f77bcf86cd799439011",
  "status": "todo",
  "priority": "high",
  "dueDate": "2026-05-01"
}

// GET /api/tasks - Get all tasks
// GET /api/tasks/user/:userId - Get user tasks
// PUT /api/tasks/:id - Update task status
{
  "status": "in-progress",
  "timeSpent": 120 // minutes
}
```

### Ticket Management APIs

```javascript
// POST /api/tickets - Create support ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when accessing admin dashboard",
  "category": "technical",
  "priority": "high"
}

// GET /api/tickets - Get all tickets
// PUT /api/tickets/:id - Update ticket
// POST /api/tickets/:id/classify - AI classify ticket
```

### AI/ML Service APIs

```javascript
// POST /ml/predict/risk - Risk prediction
{
  "userId": "507f1f77bcf86cd799439011",
  "loginAttempts": 5,
  "failedLogins": 2,
  "taskCompletionRate": 0.75,
  "averageTaskTime": 180
}

// POST /ml/detect/anomaly - Anomaly detection
{
  "userId": "507f1f77bcf86cd799439011",
  "loginTime": "2026-04-15T02:30:00Z",
  "ipAddress": "192.168.1.100",
  "location": "New York"
}

// POST /ml/detect/burnout - Burnout detection
{
  "userId": "507f1f77bcf86cd799439011",
  "weeklyHours": 65,
  "tasksCompleted": 12,
  "overdueCount": 5,
  "stressLevel": 8
}

// POST /ml/classify/ticket - Ticket classification
{
  "title": "Cannot login to system",
  "description": "Password reset not working",
  "category": "technical"
}
```

## Real Code Examples

### Frontend: User Authentication Component

```javascript
// src/components/Login.js
import React, { useState } from 'react';
import axios from 'axios';

const Login = () => {
  const [credentials, setCredentials] = useState({ email: '', password: '' });
  const [error, setError] = useState('');

  const handleLogin = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_API_URL}/api/auth/login`,
        credentials
      );
      
      // Store token and user info
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
      
      // Redirect based on role
      if (response.data.user.role === 'admin') {
        window.location.href = '/admin/dashboard';
      } else {
        window.location.href = '/user/dashboard';
      }
    } catch (err) {
      setError(err.response?.data?.message || 'Login failed');
    }
  };

  return (
    <div className="login-container">
      <form onSubmit={handleLogin}>
        <input
          type="email"
          placeholder="Email"
          value={credentials.email}
          onChange={(e) => setCredentials({ ...credentials, email: e.target.value })}
        />
        <input
          type="password"
          placeholder="Password"
          value={credentials.password}
          onChange={(e) => setCredentials({ ...credentials, password: e.target.value })}
        />
        <button type="submit">Login</button>
        {error && <p className="error">{error}</p>}
      </form>
    </div>
  );
};

export default Login;
```

### Frontend: Kanban Task Board

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [timer, setTimer] = useState(null);
  const [activeTask, setActiveTask] = useState(null);

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    
    const categorized = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    setTasks(categorized);
  };

  const moveTask = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await axios.put(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`,
      { status: newStatus },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    fetchTasks();
  };

  const startTimer = (task) => {
    setActiveTask(task);
    const startTime = Date.now();
    const interval = setInterval(() => {
      const elapsed = Math.floor((Date.now() - startTime) / 1000);
      setTimer(elapsed);
    }, 1000);
    setTimer(interval);
  };

  const stopTimer = async () => {
    if (timer && activeTask) {
      clearInterval(timer);
      const token = localStorage.getItem('token');
      await axios.put(
        `${process.env.REACT_APP_API_URL}/api/tasks/${activeTask._id}`,
        { timeSpent: (activeTask.timeSpent || 0) + Math.floor(timer / 60) },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setTimer(null);
      setActiveTask(null);
      fetchTasks();
    }
  };

  return (
    <div className="kanban-board">
      <div className="column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <button onClick={() => moveTask(task._id, 'in-progress')}>
              Start
            </button>
          </div>
        ))}
      </div>
      
      <div className="column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <button onClick={() => startTimer(task)}>⏱ Start Timer</button>
            {activeTask?._id === task._id && (
              <button onClick={stopTimer}>⏹ Stop ({timer}s)</button>
            )}
            <button onClick={() => moveTask(task._id, 'done')}>
              Complete
            </button>
          </div>
        ))}
      </div>
      
      <div className="column">
        <h3>Done</h3>
        {tasks.done.map(task => (
          <div key={task._id} className="task-card completed">
            <h4>{task.title}</h4>
            <p>Time: {task.timeSpent || 0} mins</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### Backend: User Routes with JWT Middleware

```javascript
// backend/routes/userRoutes.js
const express = require('express');
const router = express.Router();
const { protect, authorize } = require('../middleware/auth');
const User = require('../models/User');

// Protect all routes
router.use(protect);

// Get all users (Admin only)
router.get('/', authorize('admin'), async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Get single user
router.get('/:id', async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    res.json(user);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Update user
router.put('/:id', async (req, res) => {
  try {
    // Only admin or self can update
    if (req.user.role !== 'admin' && req.user.id !== req.params.id) {
      return res.status(403).json({ message: 'Not authorized' });
    }

    const { username, email, role } = req.body;
    const updateData = {};
    
    if (username) updateData.username = username;
    if (email) updateData.email = email;
    if (role && req.user.role === 'admin') updateData.role = role;

    const user = await User.findByIdAndUpdate(
      req.params.id,
      updateData,
      { new: true, runValidators: true }
    ).select('-password');

    res.json(user);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Delete user (Admin only)
router.delete('/:id', authorize('admin'), async (req, res) => {
  try {
    await User.findByIdAndDelete(req.params.id);
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = router;
```

### Backend: JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        message: `User role '${req.user.role}' is not authorized to access this route`
      });
    }
    next();
  };
};
```

### Backend: Task Management Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;
    
    const task = await Task.create({
      title,
      description,
      assignedTo,
      priority,
      dueDate,
      createdBy: req.user.id,
      status: 'todo'
    });

    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error creating task', error: error.message });
  }
};

exports.getTasks = async (req, res) => {
  try {
    const query = req.user.role === 'admin' ? {} : { assignedTo: req.user.id };
    const tasks = await Task.find(query)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username');
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching tasks', error: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const { status, timeSpent, progress } = req.body;
    
    const task = await Task.findById(req.params.id);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    if (status) task.status = status;
    if (timeSpent) task.timeSpent = (task.timeSpent || 0) + timeSpent;
    if (progress !== undefined) task.progress = progress;

    await task.save();

    // Check for potential delays using ML service
    if (status === 'in-progress') {
      try {
        const mlResponse = await axios.post(
          `${process.env.ML_SERVICE_URL}/ml/predict/delay`,
          {
            taskId: task._id,
            timeSpent: task.timeSpent,
            estimatedTime: task.estimatedTime,
            dueDate: task.dueDate
          }
        );
        
        if (mlResponse.data.delayProbability > 0.7) {
          // Send alert to admin
          console.log(`Alert: Task ${task._id} likely to be delayed`);
        }
      } catch (mlError) {
        console.error('ML service error:', mlError.message);
      }
    }

    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error updating task', error: error.message });
  }
};
```

### Backend: Ticket Management with AI Classification

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const axios = require('axios');

exports.createTicket = async (req, res) => {
  try {
    const { title, description, category, priority } = req.body;
    
    const ticket = await Ticket.create({
      title,
      description,
      category,
      priority: priority || 'medium',
      createdBy: req.user.id,
      status: 'open'
    });

    // AI-based ticket classification and routing
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/ml/classify/ticket`,
        {
          title,
          description,
          category
        }
      );

      ticket.aiCategory = mlResponse.data.category;
      ticket.aiPriority = mlResponse.data.priority;
      ticket.suggestedAssignee = mlResponse.data.suggestedAssignee;
      await ticket.save();
    } catch (mlError) {
      console.error('AI classification failed:', mlError.message);
    }

    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: 'Error creating ticket', error: error.message });
  }
};

exports.getTickets = async (req, res) => {
  try {
    const query = req.user.role === 'admin' 
      ? {} 
      : { createdBy: req.user.id };
    
    const tickets = await Ticket.find(query)
      .populate('createdBy', 'username email')
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });
    
    res.json(tickets);
  } catch (error) {
    res.status(500).json({ message: 'Error fetching tickets', error: error.message });
  }
};

exports.updateTicket = async (req, res) => {
  try {
    const { status, assignedTo, resolution } = req.body;
    
    const ticket = await Ticket.findById(req.params.id);
    if (!ticket) {
      return res.status(404).json({ message: 'Ticket not found' });
    }

    if (status) ticket.status = status;
    if (assignedTo) ticket.assignedTo = assignedTo;
    if (resolution) ticket.resolution = resolution;
    if (status === 'closed') ticket.resolvedAt = Date.now();

    await ticket.save();
    res.json(ticket);
  } catch (error) {
    res.status(500).json({ message: 'Error updating ticket', error: error.message });
  }
};
```

### ML Service: Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
from river import stream, tree, ensemble
import numpy as np
import pickle
import os

class RiskPredictor:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        self.model = None
        self.online_model = ensemble.AdaptiveRandomForestClassifier(
            n_models=10,
            seed=42
        )
        self.load_model()
    
    def load_model(self):
        """Load pre-trained model if exists"""
        if os.path.exists(self.model_path):
            with open(self.model_path, 'rb') as f:
                self.model = pickle.load(f)
        else:
            # Initialize with default model
            self.model = RandomForestClassifier(
                n_estimators=100,
                max_depth=10,
                random_state=42
            )
    
    def save_model(self):
        """Save trained model"""
        os.makedirs(os.path.dirname(self.model_path), exist_ok=True)
        with open(self.model_path, 'wb') as f:
            pickle.dump(self.model, f)
    
    def predict_risk(self, features):
        """
        Predict user risk level
        Features: [login_attempts, failed_logins, task_completion_rate, 
                   avg_task_time, overdue_tasks, weekend_logins]
        """
        features_array = np.array(features).reshape(1, -1)
        
        if self.model is None:
            return {'risk_level': 'medium', 'probability': 0.5}
        
        try:
            prediction = self.model.predict(features_array)[0]
            probability = self.model.predict_proba(features_array)[0]
            
            risk_level = 'low' if prediction == 0 else 'high'
            return {
                'risk_level': risk_level,
                'probability': float(max(probability)),
                'confidence': float(max(probability))
            }
        except Exception as e:
            print(f"Prediction error: {e}")
            return {'risk_level': 'medium', 'probability': 0.5}
    
    def update_online(self, features, label):
        """Online learning for continuous improvement"""
        feature_dict = {
            f'feature_{i}': float(v) for i, v in enumerate(features)
        }
        self.online_model.learn_one(feature_dict, int(label))
```

### ML Service: Anomaly Detection

```python
# ml-service/models/anomaly_detector.py
from sklearn.ensemble import IsolationForest
from river import anomaly
import numpy as np
from datetime import datetime

class AnomalyDetector:
    def __init__(self):
        self.isolation_forest = IsolationForest(
            contamination=0.1,
            random_state=42
        )
        self.online_detector = anomaly.HalfSpaceTrees(
            n_trees=10,
            height=8,
            seed=42
        )
        self.is_fitted = False
    
    def detect_anomaly(self, user_activity):
        """
        Detect anomalous user behavior
        user_activity: {
            'login_time': datetime,
            'ip_address': str,
            'location': str,
            'device': str,
            'actions_count': int
        }
        """
        features = self._extract_features(user_activity)
        
        # Online detection (always works)
        score = self.online_detector.score_one(features)
        self.online_detector.learn_one(features)
        
        # Threshold-based decision
        is_anomaly = score > 0.6
        
        return {
            'is_anomaly': bool(is_anomaly),
            'anomaly_score': float(score),
            'risk_factors': self._identify_risk_factors(user_activity, score)
        }
    
    def _extract_features(self, activity):
        """Extract numerical features from activity"""
        login_time = datetime.fromisoformat(activity['login_time'].replace('Z', '+00:00'))
        
        return {
            'hour_of_day': float(login_time.hour),
            'day_of_week': float(login_time.weekday()),
            'is_weekend': float(login_time.weekday() >= 5),
            'is_night': float(login_time.hour < 6 or login_time.hour > 22),
            'actions_count': float(activity.get('actions_count', 0)),
            'location_hash': float(hash(activity.get('location', '')) % 1000),
            'ip_hash': float(hash(activity.get('ip_address', '')) % 1000)
        }
    
    def _identify_risk_factors(self, activity, score):
        """Identify what makes this activity suspicious"""
        factors = []
        login_time = datetime.fromisoformat(activity['login_time'].replace('Z', '+00:00'))
        
        if login_time.hour < 6 or login_time.hour > 22:
            factors.append('unusual_login_time')
        
        if login_time.weekday() >= 5:
            factors.append('weekend_access')
        
        if activity.get('actions_count', 0) > 100:
            factors.append('high_activity_rate')
        
        return factors
```

### ML Service: Burnout Detection

```python
# ml-service/models/burnout_detector.py
import numpy as np
from sklearn.linear_model import LogisticRegression

class BurnoutDetector:
    def __init__(self):
        self.model = LogisticRegression(random_state=42)
        self.thresholds = {
            'weekly_hours': 50,
            'overdue_ratio': 0.3,
            'stress_level': 7
        }
    
    def detect_burnout(self, employee_data):
        """
        Detect employee burnout risk
        employee_data: {
            'weekly_hours': float,
            'tasks_completed': int,
            'overdue_count': int,
            'stress_level': int (1-10),
            'days_without_break': int
        }
        """
        features = [
            employee_data['weekly_hours'],
            employee_data['overdue_count'],
            employee_data.get('stress_level', 5),
            employee_data.get('days_without_break', 0),
            employee_data['tasks_completed']
        ]
        
        # Rule-based detection
        burnout_score = 0
        
        if employee_data['weekly_hours'] > self.thresholds['weekly_hours']:
            burnout_score += 0.3
        
        overdue_ratio = employee_data['overdue_count'] / max(employee_data['tasks_completed'], 1)
        if overdue_ratio > self.thresholds['overdue_ratio']:
            burnout_score += 0.3
        
        if employee_data.get('stress_level', 0) > self.thresholds['stress_level']:
            burnout_score += 0.2
        
        if employee_data.get('days_without_break', 0) > 14:
            burnout_score += 0.2
        
        risk_level = self._calculate_risk_level(burnout_score)
        
        return {
            'burnout_risk': risk_level,
            'burnout_score': float(burnout_score),
            'recommendations': self._generate_recommendations(employee_data, burnout_score)
        }
    
    def _calculate_risk_level(self, score):
        """Convert score to risk level"""
        if score < 0.3:
            return 'low'
        elif score < 0.6:
            return 'medium'
        else:
            return 'high'
    
    def _generate_recommendations(self, data, score):
        """Generate actionable recommendations"""
        recommendations = []
        
        if data['weekly_hours'] > 50:
            recommendations.append('Reduce weekly working hours')
        
        if data['overdue_count'] > 3:
            recommendations.append('Redistribute workload or extend deadlines')
        
        if data.get('days_without_break', 0) > 14:
            recommendations.append('Schedule time off')
        
        if data.get('stress_level', 0) > 7:
            recommendations.append('Provide stress management resources')
        
        return recommendations
```

### ML Service: FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
from models.risk_predictor import RiskPredictor
from models.anomaly_detector import AnomalyDetector
from models.burnout_detector import BurnoutDetector
import os

app = FastAPI(title="Enterprise UMS ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize models
risk_predictor = RiskPredictor()
anomaly_detector = AnomalyDetector()
burnout_detector = BurnoutDetector()

# Request models
class RiskPredictionRequest(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    taskCompletionRate: float
    averageTaskTime: float
    overdueTasks: Optional[int] = 0
    weekendLogins: Optional[int
