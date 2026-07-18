---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "add AI-powered ticket classification"
  - "build admin panel with role-based access"
  - "integrate anomaly detection and burnout analysis"
  - "deploy user management system with ML service"
  - "configure JWT authentication for user system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack application that combines user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication, and user profiles
- **Task Tracking**: Kanban boards, time tracking, and progress monitoring
- **Support Tickets**: Smart ticket classification and routing
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, and predictive insights
- **Admin Dashboard**: Centralized monitoring and audit logs

The system consists of three main components:
1. **Frontend**: React.js application
2. **Backend**: Node.js REST API with MongoDB
3. **ML Service**: FastAPI service with scikit-learn and River for online learning

## Installation

### Prerequisites

- Node.js (v14+)
- Python (v3.8+)
- MongoDB (running instance)

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
MONGODB_URI=mongodb://localhost:27017/user_management
JWT_SECRET=your_jwt_secret_here
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
MONGODB_URI=mongodb://localhost:27017/user_management
MODEL_PATH=./models
LOG_LEVEL=info
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

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Key API Endpoints

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
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
    "id": "user_id",
    "email": "john@example.com",
    "role": "user"
  }
}
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Create user
POST /api/users
{
  "username": "jane.smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
{
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:userId
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks?userId=<userId>

// Create task
POST /api/tasks
{
  "title": "Implement user authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo",
  "dueDate": "2026-05-01"
}

// Update task status
PUT /api/tasks/:taskId
{
  "status": "in_progress",
  "timeSpent": 120
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Unable to login",
  "description": "Getting authentication error",
  "priority": "high",
  "userId": "user_id"
}

// AI classification response
{
  "ticketId": "ticket_id",
  "category": "authentication",
  "suggestedAssignee": "support_team_id",
  "priority": "high"
}
```

### AI Analytics Endpoints

```javascript
// Risk prediction
POST /api/ml/predict-risk
{
  "userId": "user_id",
  "behaviorData": {
    "loginAttempts": 5,
    "failedLogins": 3,
    "accessPatterns": ["unusual_time", "new_location"]
  }
}

// Response
{
  "riskScore": 0.75,
  "riskLevel": "high",
  "factors": ["multiple_failed_logins", "unusual_access_pattern"]
}

// Anomaly detection
POST /api/ml/detect-anomaly
{
  "userId": "user_id",
  "activityLog": [
    {"action": "login", "timestamp": "2026-04-15T02:30:00Z"},
    {"action": "data_access", "resource": "sensitive_db"}
  ]
}

// Burnout analysis
POST /api/ml/analyze-burnout
{
  "userId": "user_id",
  "workloadData": {
    "tasksCompleted": 45,
    "averageWorkHours": 11.5,
    "overtimeHours": 20,
    "taskCompletionRate": 0.65
  }
}

// Response
{
  "burnoutScore": 0.82,
  "burnoutRisk": "high",
  "recommendations": [
    "Reduce workload",
    "Schedule time off",
    "Redistribute tasks"
  ]
}
```

## Frontend React Integration

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
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
      const response = await axios.get(`${API_URL}/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/auth/login`, {
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

  return { user, loading, login, logout };
};
```

### Task Dashboard Component

```javascript
// src/components/TaskDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const TaskDashboard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/tasks?userId=${userId}`);
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(categorized);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`${API_URL}/tasks/${taskId}`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(column => (
        <div key={column} className="kanban-column">
          <h3>{column.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          {tasks[column].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-${task.priority}`}>
                {task.priority}
              </span>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default TaskDashboard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_URL = process.env.REACT_APP_ML_URL;

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutScore: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch risk score
      const riskResponse = await axios.post(`${ML_URL}/predict-risk`, {
        userId
      });

      // Fetch burnout analysis
      const burnoutResponse = await axios.post(`${ML_URL}/analyze-burnout`, {
        userId
      });

      setAnalytics({
        riskScore: riskResponse.data.riskScore,
        burnoutScore: burnoutResponse.data.burnoutScore,
        anomalies: riskResponse.data.anomalies || []
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      <div className="metric-card">
        <h3>Risk Score</h3>
        <div className={`score ${analytics.riskScore > 0.7 ? 'high' : 'low'}`}>
          {(analytics.riskScore * 100).toFixed(1)}%
        </div>
      </div>

      <div className="metric-card">
        <h3>Burnout Risk</h3>
        <div className={`score ${analytics.burnoutScore > 0.7 ? 'high' : 'low'}`}>
          {(analytics.burnoutScore * 100).toFixed(1)}%
        </div>
      </div>

      {analytics.anomalies.length > 0 && (
        <div className="anomalies">
          <h3>Detected Anomalies</h3>
          <ul>
            {analytics.anomalies.map((anomaly, idx) => (
              <li key={idx}>{anomaly.description}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

## Backend API Implementation Patterns

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Get all users (Admin only)
exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

// Create user
exports.createUser = async (req, res) => {
  try {
    const { username, email, password, role, department } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const user = new User({
      username,
      email,
      password,
      role: role || 'user',
      department
    });

    await user.save();
    
    res.status(201).json({
      message: 'User created successfully',
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
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const { userId } = req.params;
    const updates = req.body;

    // Don't allow password update through this endpoint
    delete updates.password;

    const user = await User.findByIdAndUpdate(
      userId,
      { $set: updates },
      { new: true, runValidators: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json({ message: 'User updated successfully', user });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

// Get tasks with filters
exports.getTasks = async (req, res) => {
  try {
    const { userId, status, priority } = req.query;
    
    const filter = {};
    if (userId) filter.assignedTo = userId;
    if (status) filter.status = status;
    if (priority) filter.priority = priority;

    const tasks = await Task.find(filter)
      .populate('assignedTo', 'username email')
      .sort({ createdAt: -1 });

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

// Create task
exports.createTask = async (req, res) => {
  try {
    const task = new Task(req.body);
    await task.save();
    
    const populatedTask = await Task.findById(task._id)
      .populate('assignedTo', 'username email');

    res.status(201).json({
      message: 'Task created successfully',
      task: populatedTask
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

// Update task with time tracking
exports.updateTask = async (req, res) => {
  try {
    const { taskId } = req.params;
    const { status, timeSpent } = req.body;

    const updateData = { ...req.body };
    
    // Track time spent
    if (timeSpent) {
      const task = await Task.findById(taskId);
      updateData.totalTimeSpent = (task.totalTimeSpent || 0) + timeSpent;
    }

    const task = await Task.findByIdAndUpdate(
      taskId,
      { $set: updateData },
      { new: true }
    ).populate('assignedTo', 'username email');

    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    res.json({ message: 'Task updated successfully', task });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId).select('-password');

    if (!user) {
      return res.status(401).json({ message: 'User not found' });
    }

    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

// Role-based access control
exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: 'Access denied. Insufficient permissions.' 
      });
    }
    next();
  };
};
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier, IsolationForest
import joblib
import os

app = FastAPI(title="User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Models
risk_model = None
anomaly_detector = None

class RiskPredictionRequest(BaseModel):
    userId: str
    behaviorData: Dict

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    workloadData: Dict

class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    userId: str

@app.on_event("startup")
async def load_models():
    global risk_model, anomaly_detector
    
    # Try to load existing models or create new ones
    try:
        risk_model = joblib.load(f'{MODEL_PATH}/risk_model.pkl')
    except:
        risk_model = RandomForestClassifier(n_estimators=100)
    
    try:
        anomaly_detector = joblib.load(f'{MODEL_PATH}/anomaly_detector.pkl')
    except:
        anomaly_detector = IsolationForest(contamination=0.1, random_state=42)

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavior patterns"""
    try:
        # Extract features
        features = extract_risk_features(request.behaviorData)
        
        # Calculate risk score
        risk_score = calculate_risk_score(features)
        
        # Determine risk level
        risk_level = "high" if risk_score > 0.7 else "medium" if risk_score > 0.4 else "low"
        
        # Identify risk factors
        factors = identify_risk_factors(features)
        
        return {
            "userId": request.userId,
            "riskScore": float(risk_score),
            "riskLevel": risk_level,
            "factors": factors,
            "timestamp": "2026-04-15T17:20:35Z"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(activityLog: List[Dict]):
    """Detect anomalies in user activity"""
    try:
        # Extract features from activity log
        features = extract_activity_features(activityLog)
        
        # Reshape for model
        X = np.array(features).reshape(1, -1)
        
        # Predict anomaly (-1 = anomaly, 1 = normal)
        prediction = anomaly_detector.predict(X)[0]
        anomaly_score = anomaly_detector.score_samples(X)[0]
        
        is_anomaly = prediction == -1
        
        return {
            "isAnomaly": bool(is_anomaly),
            "anomalyScore": float(anomaly_score),
            "confidence": abs(float(anomaly_score)),
            "details": analyze_anomaly(activityLog) if is_anomaly else None
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze employee burnout risk"""
    try:
        workload = request.workloadData
        
        # Calculate burnout indicators
        workload_score = min(workload.get('averageWorkHours', 8) / 12, 1.0)
        overtime_score = min(workload.get('overtimeHours', 0) / 20, 1.0)
        completion_score = 1 - workload.get('taskCompletionRate', 1.0)
        task_volume_score = min(workload.get('tasksCompleted', 20) / 50, 1.0)
        
        # Weighted burnout score
        burnout_score = (
            workload_score * 0.3 +
            overtime_score * 0.3 +
            completion_score * 0.2 +
            task_volume_score * 0.2
        )
        
        # Determine risk level
        burnout_risk = "high" if burnout_score > 0.7 else "medium" if burnout_score > 0.4 else "low"
        
        # Generate recommendations
        recommendations = generate_burnout_recommendations(burnout_score, workload)
        
        return {
            "userId": request.userId,
            "burnoutScore": float(burnout_score),
            "burnoutRisk": burnout_risk,
            "recommendations": recommendations,
            "metrics": {
                "workloadScore": float(workload_score),
                "overtimeScore": float(overtime_score),
                "completionScore": float(completion_score)
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket and suggest routing"""
    try:
        # Simple keyword-based classification
        text = (request.title + " " + request.description).lower()
        
        categories = {
            "authentication": ["login", "password", "access", "authentication"],
            "technical": ["bug", "error", "crash", "not working"],
            "account": ["account", "profile", "settings"],
            "performance": ["slow", "performance", "loading"]
        }
        
        category = "general"
        for cat, keywords in categories.items():
            if any(keyword in text for keyword in keywords):
                category = cat
                break
        
        # Priority determination
        urgent_keywords = ["urgent", "critical", "emergency", "asap"]
        priority = "high" if any(kw in text for kw in urgent_keywords) else "medium"
        
        return {
            "category": category,
            "priority": priority,
            "suggestedAssignee": get_assignee_for_category(category),
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# Helper functions
def extract_risk_features(behavior_data: Dict) -> np.ndarray:
    """Extract numerical features from behavior data"""
    return np.array([
        behavior_data.get('loginAttempts', 0),
        behavior_data.get('failedLogins', 0),
        len(behavior_data.get('accessPatterns', [])),
        1 if 'unusual_time' in behavior_data.get('accessPatterns', []) else 0,
        1 if 'new_location' in behavior_data.get('accessPatterns', []) else 0
    ])

def calculate_risk_score(features: np.ndarray) -> float:
    """Calculate risk score from features"""
    # Simple heuristic-based scoring
    login_attempts, failed_logins, pattern_count, unusual_time, new_location = features
    
    score = 0.0
    score += min(failed_logins / 5, 0.4)  # Up to 40% for failed logins
    score += min(pattern_count / 3, 0.3)  # Up to 30% for unusual patterns
    score += unusual_time * 0.15          # 15% for unusual time
    score += new_location * 0.15          # 15% for new location
    
    return min(score, 1.0)

def identify_risk_factors(features: np.ndarray) -> List[str]:
    """Identify specific risk factors"""
    factors = []
    login_attempts, failed_logins, pattern_count, unusual_time, new_location = features
    
    if failed_logins >= 3:
        factors.append("multiple_failed_logins")
    if unusual_time:
        factors.append("unusual_access_time")
    if new_location:
        factors.append("new_location_access")
    if pattern_count >= 2:
        factors.append("unusual_access_pattern")
    
    return factors

def extract_activity_features(activity_log: List[Dict]) -> List[float]:
    """Extract features from activity log"""
    if not activity_log:
        return [0] * 5
    
    return [
        len(activity_log),  # Number of activities
        len(set(a.get('action') for a in activity_log)),  # Unique actions
        sum(1 for a in activity_log if 'sensitive' in a.get('resource', '')),  # Sensitive access
        len([a for a in activity_log if a.get('action') == 'login']),  # Login count
        len([a for a in activity_log if a.get('action') == 'data_access'])  # Data access count
    ]

def analyze_anomaly(activity_log: List[Dict]) -> Dict:
    """Analyze detected anomaly"""
    return {
        "reason": "Unusual activity pattern detected",
        "activities": len(activity_log),
        "recommendation": "Review user activity and verify legitimacy"
    }

def generate_burnout_recommendations(score: float, workload: Dict) -> List[str]:
    """Generate burnout prevention recommendations"""
    recommendations = []
    
    if workload.get('averageWorkHours', 8) > 10:
        recommendations.append("Reduce daily work hours to sustainable levels")
    
    if workload.get('overtimeHours', 0) > 15:
        recommendations.append("Schedule time off to recover")
    
    if workload.get('taskCompletionRate', 1.0) < 0.7:
        recommendations.append("Redistribute tasks to balance workload")
    
    if score > 0.7:
        recommendations.append("Schedule meeting with manager to discuss workload")
    
    if not recommendations:
        recommendations.append("Continue current pace with regular breaks")
    
    return recommendations

def get_assignee_for_category(category: str) -> str:
    """Get suggested assignee based on ticket category"""
    assignee_map = {
        "authentication": "security_team",
        "technical": "engineering_team",
        "account": "support_team",
        "performance": "engineering_team",
        "general": "support_team"
    }
    return assignee_map.get(category, "support_team")

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Database Models

### User Model (MongoDB with Mongoose)

```javascript
// backend/models/User.js
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
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  department: {
    type: String,
    required: false
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  lastLogin: {
    type: Date
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
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
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
