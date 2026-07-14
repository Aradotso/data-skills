---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create admin dashboard with user tracking"
  - "build task management system with kanban board"
  - "integrate AI risk detection and anomaly detection"
  - "develop support ticket system with AI classification"
  - "add burnout detection to user management"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill provides expertise in using and extending the Enterprise User Management System, a full-stack application that combines user/task management with AI-driven analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Role-based access control (Admin/User), authentication with JWT
- **Task Management**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Real-time insights for admins and personalized dashboards for users

**Architecture**: React frontend, Node.js/Express backend, FastAPI ML service, MongoDB database

## Installation

### Prerequisites

- Node.js 16+ and npm
- Python 3.8+
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

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
EOF

# Start backend
npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file if needed
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=info
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

ML service runs at `http://localhost:8000`

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

Frontend runs at `http://localhost:3000`

## Key API Endpoints

### Authentication (Backend)

```javascript
// POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "securepass123",
  "role": "user" // or "admin"
}

// POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securepass123"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": { "id": "...", "email": "...", "role": "user" }
}
```

### User Management (Backend)

```javascript
// GET /api/users - Get all users (admin only)
// GET /api/users/:id - Get specific user
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user (admin only)

// Example: Update user
fetch('http://localhost:5000/api/users/123', {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    username: "john.doe.updated",
    email: "john.updated@example.com"
  })
});
```

### Task Management (Backend)

```javascript
// POST /api/tasks - Create task
{
  "title": "Implement login feature",
  "description": "Add JWT authentication",
  "assignedTo": "user_id",
  "status": "todo", // todo, inprogress, done
  "priority": "high",
  "dueDate": "2026-05-01"
}

// GET /api/tasks - Get all tasks
// GET /api/tasks/user/:userId - Get user tasks
// PUT /api/tasks/:id - Update task status
// DELETE /api/tasks/:id - Delete task
```

### Support Tickets (Backend)

```javascript
// POST /api/tickets - Create ticket
{
  "title": "Cannot login to system",
  "description": "Getting 401 error",
  "priority": "high",
  "userId": "user_id"
}

// GET /api/tickets - Get all tickets
// PUT /api/tickets/:id - Update ticket
{
  "status": "resolved",
  "response": "Fixed authentication issue"
}
```

### AI Analytics (ML Service)

```python
# POST /api/ai/risk-prediction
{
  "userId": "user_id",
  "features": {
    "login_attempts": 5,
    "tasks_completed": 10,
    "tickets_raised": 3,
    "avg_task_completion_time": 2.5
  }
}

# Response
{
  "risk_level": "low",
  "risk_score": 0.23,
  "recommendations": ["Continue monitoring"]
}

# POST /api/ai/anomaly-detection
{
  "userId": "user_id",
  "activity": {
    "login_time": "2026-04-15T23:30:00",
    "location": "unusual_ip",
    "actions": ["bulk_download", "access_admin_panel"]
  }
}

# Response
{
  "is_anomaly": true,
  "anomaly_score": 0.87,
  "detected_patterns": ["unusual_time", "suspicious_actions"]
}

# POST /api/ai/burnout-detection
{
  "userId": "user_id",
  "workload": {
    "tasks_assigned": 25,
    "avg_hours_per_day": 11,
    "days_without_break": 14,
    "overdue_tasks": 8
  }
}

# Response
{
  "burnout_risk": "high",
  "burnout_score": 0.82,
  "recommendations": [
    "Reduce workload by 30%",
    "Schedule mandatory break"
  ]
}

# POST /api/ai/predict-delay
{
  "projectId": "proj_123",
  "metrics": {
    "total_tasks": 50,
    "completed_tasks": 15,
    "days_elapsed": 30,
    "planned_days": 90,
    "team_size": 5
  }
}

# Response
{
  "delay_predicted": true,
  "estimated_delay_days": 15,
  "completion_probability": 0.68,
  "recommendations": ["Add 2 team members", "Reduce scope"]
}
```

## Real Code Examples

### Frontend: User Authentication Flow

```javascript
// src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const authService = {
  async login(email, password) {
    try {
      const response = await axios.post(`${API_URL}/api/auth/login`, {
        email,
        password
      });
      
      if (response.data.token) {
        localStorage.setItem('user', JSON.stringify(response.data));
      }
      
      return response.data;
    } catch (error) {
      throw error.response?.data?.message || 'Login failed';
    }
  },

  logout() {
    localStorage.removeItem('user');
  },

  getCurrentUser() {
    return JSON.parse(localStorage.getItem('user'));
  },

  getAuthHeader() {
    const user = this.getCurrentUser();
    if (user && user.token) {
      return { Authorization: `Bearer ${user.token}` };
    }
    return {};
  }
};

// src/components/Login.jsx
import React, { useState } from 'react';
import { authService } from '../services/authService';
import { useNavigate } from 'react-router-dom';

export const Login = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const navigate = useNavigate();

  const handleSubmit = async (e) => {
    e.preventDefault();
    setError('');

    try {
      const user = await authService.login(email, password);
      
      if (user.role === 'admin') {
        navigate('/admin/dashboard');
      } else {
        navigate('/user/dashboard');
      }
    } catch (err) {
      setError(err);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
        required
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
        required
      />
      {error && <div className="error">{error}</div>}
      <button type="submit">Login</button>
    </form>
  );
};
```

### Frontend: Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { authService } from '../services/authService';

export const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inprogress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
        { headers: authService.getAuthHeader() }
      );
      
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status].push(task);
        return acc;
      }, { todo: [], inprogress: [], done: [] });
      
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');

    try {
      await axios.put(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`,
        { status: newStatus },
        { headers: authService.getAuthHeader() }
      );
      
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  const renderColumn = (status, title) => (
    <div
      className="kanban-column"
      onDrop={(e) => handleDrop(e, status)}
      onDragOver={handleDragOver}
    >
      <h3>{title}</h3>
      {tasks[status].map(task => (
        <div
          key={task._id}
          className="task-card"
          draggable
          onDragStart={(e) => handleDragStart(e, task._id)}
        >
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority-${task.priority}`}>
            {task.priority}
          </span>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do')}
      {renderColumn('inprogress', 'In Progress')}
      {renderColumn('done', 'Done')}
    </div>
  );
};
```

### Backend: Task API with JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ message: 'No token provided' });
  }

  try {
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

// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const Task = require('../models/Task');
const { authMiddleware, adminMiddleware } = require('../middleware/auth');

// Create task (admin only)
router.post('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// Get user tasks
router.get('/user/:userId', authMiddleware, async (req, res) => {
  try {
    // Users can only see their own tasks unless admin
    if (req.user.id !== req.params.userId && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Unauthorized' });
    }

    const tasks = await Task.find({ assignedTo: req.params.userId })
      .sort({ createdAt: -1 });
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

// Update task
router.put('/:id', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    // Users can update their own tasks, admins can update any
    if (task.assignedTo.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Unauthorized' });
    }

    Object.assign(task, req.body);
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

module.exports = router;

// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String, required: true },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
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
  dueDate: { type: Date },
  timeTracked: { type: Number, default: 0 }, // minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

taskSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

module.exports = mongoose.model('Task', taskSchema);
```

### ML Service: AI Analytics Implementation

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Dict, List
import numpy as np
from sklearn.ensemble import IsolationForest, RandomForestClassifier
from river import anomaly, drift
import joblib
import os

app = FastAPI()

# Load or initialize models
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

class RiskPredictionRequest(BaseModel):
    userId: str
    features: Dict[str, float]

class AnomalyDetectionRequest(BaseModel):
    userId: str
    activity: Dict[str, any]

class BurnoutDetectionRequest(BaseModel):
    userId: str
    workload: Dict[str, float]

class DelayPredictionRequest(BaseModel):
    projectId: str
    metrics: Dict[str, float]

# Risk Prediction Model
class RiskPredictor:
    def __init__(self):
        model_file = f"{MODEL_PATH}/risk_model.pkl"
        if os.path.exists(model_file):
            self.model = joblib.load(model_file)
        else:
            self.model = RandomForestClassifier(n_estimators=100)
            # Initialize with dummy data (in production, train with real data)
            X = np.random.rand(100, 4)
            y = np.random.randint(0, 3, 100)  # 0=low, 1=medium, 2=high
            self.model.fit(X, y)
            joblib.dump(self.model, model_file)
    
    def predict(self, features: Dict[str, float]) -> Dict:
        feature_vector = np.array([
            features.get('login_attempts', 0),
            features.get('tasks_completed', 0),
            features.get('tickets_raised', 0),
            features.get('avg_task_completion_time', 0)
        ]).reshape(1, -1)
        
        prediction = self.model.predict(feature_vector)[0]
        proba = self.model.predict_proba(feature_vector)[0]
        
        risk_levels = ['low', 'medium', 'high']
        recommendations = self._generate_recommendations(prediction, features)
        
        return {
            'risk_level': risk_levels[prediction],
            'risk_score': float(proba[prediction]),
            'recommendations': recommendations
        }
    
    def _generate_recommendations(self, risk_level: int, features: Dict) -> List[str]:
        recommendations = []
        
        if features.get('login_attempts', 0) > 5:
            recommendations.append('High login attempts detected - verify credentials')
        
        if features.get('tickets_raised', 0) > 5:
            recommendations.append('High ticket count - provide additional training')
        
        if features.get('avg_task_completion_time', 0) > 5:
            recommendations.append('Slow task completion - review workload')
        
        if not recommendations:
            recommendations.append('Continue monitoring')
        
        return recommendations

# Anomaly Detection
class AnomalyDetector:
    def __init__(self):
        self.model = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)
    
    def detect(self, activity: Dict) -> Dict:
        # Convert activity to feature vector
        features = {
            'hour': self._extract_hour(activity.get('login_time', '')),
            'is_unusual_location': 1 if 'unusual' in str(activity.get('location', '')).lower() else 0,
            'action_count': len(activity.get('actions', [])),
            'has_suspicious_action': 1 if any('admin' in str(a).lower() or 'bulk' in str(a).lower() 
                                             for a in activity.get('actions', [])) else 0
        }
        
        feature_vector = features
        score = self.model.score_one(feature_vector)
        self.model.learn_one(feature_vector)
        
        is_anomaly = score > 0.7
        detected_patterns = []
        
        if features['hour'] < 6 or features['hour'] > 22:
            detected_patterns.append('unusual_time')
        if features['is_unusual_location']:
            detected_patterns.append('unusual_location')
        if features['has_suspicious_action']:
            detected_patterns.append('suspicious_actions')
        
        return {
            'is_anomaly': is_anomaly,
            'anomaly_score': float(score),
            'detected_patterns': detected_patterns
        }
    
    def _extract_hour(self, timestamp: str) -> int:
        try:
            from datetime import datetime
            dt = datetime.fromisoformat(timestamp.replace('Z', '+00:00'))
            return dt.hour
        except:
            return 12  # default to noon

# Burnout Detection
class BurnoutDetector:
    def detect(self, workload: Dict[str, float]) -> Dict:
        score = 0.0
        
        # Calculate burnout score based on multiple factors
        tasks_assigned = workload.get('tasks_assigned', 0)
        if tasks_assigned > 20:
            score += 0.3
        
        avg_hours = workload.get('avg_hours_per_day', 0)
        if avg_hours > 10:
            score += 0.3
        elif avg_hours > 8:
            score += 0.15
        
        days_without_break = workload.get('days_without_break', 0)
        if days_without_break > 10:
            score += 0.25
        
        overdue_tasks = workload.get('overdue_tasks', 0)
        if overdue_tasks > 5:
            score += 0.15
        
        # Determine risk level
        if score > 0.7:
            risk = 'high'
        elif score > 0.4:
            risk = 'medium'
        else:
            risk = 'low'
        
        recommendations = self._generate_burnout_recommendations(workload, score)
        
        return {
            'burnout_risk': risk,
            'burnout_score': float(score),
            'recommendations': recommendations
        }
    
    def _generate_burnout_recommendations(self, workload: Dict, score: float) -> List[str]:
        recommendations = []
        
        if workload.get('tasks_assigned', 0) > 20:
            recommendations.append('Reduce workload by 30%')
        
        if workload.get('avg_hours_per_day', 0) > 10:
            recommendations.append('Limit working hours to 8 per day')
        
        if workload.get('days_without_break', 0) > 10:
            recommendations.append('Schedule mandatory break')
        
        if workload.get('overdue_tasks', 0) > 5:
            recommendations.append('Reassign overdue tasks')
        
        return recommendations

# Delay Prediction
class DelayPredictor:
    def predict(self, metrics: Dict[str, float]) -> Dict:
        total_tasks = metrics.get('total_tasks', 0)
        completed_tasks = metrics.get('completed_tasks', 0)
        days_elapsed = metrics.get('days_elapsed', 0)
        planned_days = metrics.get('planned_days', 0)
        team_size = metrics.get('team_size', 1)
        
        # Calculate progress rate
        completion_rate = completed_tasks / total_tasks if total_tasks > 0 else 0
        expected_completion_rate = days_elapsed / planned_days if planned_days > 0 else 0
        
        # Simple prediction logic
        if completion_rate < expected_completion_rate - 0.2:
            delay_predicted = True
            delay_days = int((1 - completion_rate) * planned_days * 0.3)
            completion_probability = 0.5
        elif completion_rate < expected_completion_rate:
            delay_predicted = True
            delay_days = int((1 - completion_rate) * planned_days * 0.15)
            completion_probability = 0.7
        else:
            delay_predicted = False
            delay_days = 0
            completion_probability = 0.9
        
        recommendations = []
        if delay_predicted:
            if team_size < 5:
                recommendations.append(f'Add {max(1, 5 - team_size)} team members')
            if completion_rate < 0.5:
                recommendations.append('Reduce scope')
            recommendations.append('Schedule daily standups')
        
        return {
            'delay_predicted': delay_predicted,
            'estimated_delay_days': delay_days,
            'completion_probability': completion_probability,
            'recommendations': recommendations
        }

# Initialize models
risk_predictor = RiskPredictor()
anomaly_detector = AnomalyDetector()
burnout_detector = BurnoutDetector()
delay_predictor = DelayPredictor()

# API Endpoints
@app.post('/api/ai/risk-prediction')
async def predict_risk(request: RiskPredictionRequest):
    try:
        result = risk_predictor.predict(request.features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post('/api/ai/anomaly-detection')
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        result = anomaly_detector.detect(request.activity)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post('/api/ai/burnout-detection')
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        result = burnout_detector.detect(request.workload)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post('/api/ai/predict-delay')
async def predict_delay(request: DelayPredictionRequest):
    try:
        result = delay_predictor.predict(request.metrics)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get('/health')
async def health_check():
    return {'status': 'healthy', 'service': 'ML Analytics'}
```

### Frontend: AI Analytics Dashboard

```javascript
// src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { authService } from '../services/authService';

export const AIAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    risk: null,
    burnout: null,
    anomalies: []
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    setLoading(true);
    try {
      // Fetch user data first
      const userResponse = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/users/${userId}`,
        { headers: authService.getAuthHeader() }
      );

      const userData = userResponse.data;

      // Get risk prediction
      const riskResponse = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/ai/risk-prediction`,
        {
          userId: userId,
          features: {
            login_attempts: userData.loginAttempts || 0,
            tasks_completed: userData.tasksCompleted || 0,
            tickets_raised: userData.ticketsRaised || 0,
            avg_task_completion_time: userData.avgCompletionTime || 0
          }
        }
      );

      // Get burnout detection
      const burnoutResponse = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/ai/burnout-detection`,
        {
          userId: userId,
          workload: {
            tasks_assigned: userData.tasksAssigned || 0,
            avg_hours_per_day: userData.avgHoursPerDay || 8,
            days_without_break: userData.daysWithoutBreak || 0,
            overdue_tasks: userData.overdueTasks || 0
          }
        }
      );

      setAnalytics({
        risk: riskResponse.data,
        burnout: burnoutResponse.data,
        anomalies: []
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) {
    return <div>Loading analytics...</div>;
  }

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI Analytics</h2>

      {/* Risk Assessment */}
      <div className={`card risk-${analytics.risk?.risk_level}`}>
        <h3>Risk Assessment</h3>
        <div className="risk-level">
          Level: {analytics.risk?.risk_level?.toUpperCase()}
        </div>
        <div className="risk-score">
          Score: {(analytics.risk?.risk_score * 100).toFixed(1)}%
        </div>
        <div className="recommendations">
          <h4>Recommendations:</h4>
          <ul>
            {analytics.risk?.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      </div>

      {/* Burnout Detection */}
      <div className={`card burnout-${analytics.burnout?.burnout_risk}`}>
        <h3>Burnout Detection</h3>
        <div className="burnout-risk">
          Risk: {analytics.burnout?.burnout_risk?.toUpperCase()}
        </div>
        <div className="burnout-score">
          Score: {(analytics.burnout?.burnout_score * 100).toFixed(1)}%
        </div>
        <div className="recommendations">
          <h4>Recommendations:</h4>
          <ul>
            {analytics.burnout?.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        </div>
      </div>

      <button onClick
