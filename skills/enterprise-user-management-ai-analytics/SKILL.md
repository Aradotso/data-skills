---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered task analytics, risk detection, and automated ticket routing
triggers:
  - "set up enterprise user management with AI analytics"
  - "implement AI-powered user management system"
  - "create task management with burnout detection"
  - "build admin dashboard with AI insights"
  - "add AI ticket classification and routing"
  - "integrate risk prediction for user management"
  - "develop kanban board with time tracking"
  - "implement JWT authentication for user system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise application for managing users, tasks, and support tickets with integrated AI analytics for risk detection, anomaly detection, burnout analysis, and predictive project insights. Built with React, Node.js, MongoDB, and FastAPI ML service.

## What This Project Does

This system provides:
- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, user monitoring
- **User Dashboard**: Personal task overview, performance insights, notifications

## Installation

### Prerequisites
- Node.js 14+
- Python 3.8+
- MongoDB

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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
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
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Architecture Overview

```
frontend/          # React.js UI
├── src/
│   ├── components/
│   ├── pages/
│   ├── services/
│   └── utils/

backend/           # Node.js REST API
├── routes/
├── controllers/
├── models/
├── middleware/
└── config/

ml-service/        # FastAPI ML endpoints
├── models/
├── services/
└── main.py
```

## Key API Endpoints

### Authentication

```javascript
// Login
POST /api/auth/login
Body: { email, password }
Response: { token, user: { id, name, email, role } }

// Register
POST /api/auth/register
Body: { name, email, password, role }
Response: { token, user }
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer <token>" }
Response: [{ id, name, email, role, status, createdAt }]

// Create user
POST /api/users
Body: { name, email, password, role, department }

// Update user
PUT /api/users/:id
Body: { name, email, role, status }

// Delete user
DELETE /api/users/:id
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks/user/:userId
Response: [{ id, title, description, status, priority, assignedTo, dueDate }]

// Create task
POST /api/tasks
Body: { title, description, assignedTo, priority, dueDate, estimatedHours }

// Update task status
PUT /api/tasks/:id/status
Body: { status: "todo" | "in-progress" | "done" }

// Track time
POST /api/tasks/:id/time
Body: { duration, startTime, endTime }
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Body: { title, description, priority, category }

// Get tickets
GET /api/tickets
Query: ?status=open&priority=high

// AI classify ticket
POST /api/ml/classify-ticket
Body: { title, description }
Response: { category, priority, assignedTo }
```

### AI Analytics

```javascript
// Risk prediction
POST /api/ml/predict-risk
Body: { userId, recentActivity, taskCompletionRate }
Response: { riskScore, factors, recommendations }

// Anomaly detection
POST /api/ml/detect-anomaly
Body: { userId, loginTimes, accessPatterns }
Response: { isAnomaly, anomalyScore, details }

// Burnout analysis
POST /api/ml/burnout-analysis
Body: { userId, workHours, taskLoad, completionRate }
Response: { burnoutScore, factors, suggestions }

// Project delay prediction
POST /api/ml/predict-delay
Body: { projectId, tasks, teamSize, deadline }
Response: { delayProbability, estimatedDelay, riskFactors }
```

## React Frontend Examples

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
      const response = await axios.get(`${API_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/api/auth/login`, { email, password });
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

### Task Management Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks/user/${userId}`);
      const grouped = response.data.reduce((acc, task) => {
        const status = task.status === 'in-progress' ? 'inProgress' : task.status;
        acc[status] = [...(acc[status] || []), task];
        return acc;
      }, { todo: [], inProgress: [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`${API_URL}/api/tasks/${taskId}/status`, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const TaskColumn = ({ title, status, taskList }) => (
    <div className="task-column">
      <h3>{title}</h3>
      {taskList.map(task => (
        <div key={task.id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority-${task.priority}`}>{task.priority}</span>
          {status !== 'done' && (
            <button onClick={() => updateTaskStatus(task.id, 
              status === 'todo' ? 'in-progress' : 'done'
            )}>
              {status === 'todo' ? 'Start' : 'Complete'}
            </button>
          )}
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      <TaskColumn title="To Do" status="todo" taskList={tasks.todo} />
      <TaskColumn title="In Progress" status="inProgress" taskList={tasks.inProgress} />
      <TaskColumn title="Done" status="done" taskList={tasks.done} />
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIInsights.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const AIInsights = ({ userId }) => {
  const [insights, setInsights] = useState({
    risk: null,
    burnout: null,
    anomalies: null
  });

  useEffect(() => {
    fetchAIInsights();
  }, [userId]);

  const fetchAIInsights = async () => {
    try {
      const [riskRes, burnoutRes, anomalyRes] = await Promise.all([
        axios.post(`${ML_API_URL}/api/ml/predict-risk`, { userId }),
        axios.post(`${ML_API_URL}/api/ml/burnout-analysis`, { userId }),
        axios.post(`${ML_API_URL}/api/ml/detect-anomaly`, { userId })
      ]);

      setInsights({
        risk: riskRes.data,
        burnout: burnoutRes.data,
        anomalies: anomalyRes.data
      });
    } catch (error) {
      console.error('Failed to fetch AI insights:', error);
    }
  };

  return (
    <div className="ai-insights">
      <h2>AI-Powered Insights</h2>
      
      {insights.risk && (
        <div className="insight-card">
          <h3>Risk Assessment</h3>
          <div className={`risk-score score-${Math.floor(insights.risk.riskScore / 25)}`}>
            {insights.risk.riskScore}%
          </div>
          <ul>
            {insights.risk.recommendations.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        </div>
      )}

      {insights.burnout && (
        <div className="insight-card">
          <h3>Burnout Analysis</h3>
          <div className="burnout-score">
            Score: {insights.burnout.burnoutScore}/100
          </div>
          <ul>
            {insights.burnout.suggestions.map((sug, i) => (
              <li key={i}>{sug}</li>
            ))}
          </ul>
        </div>
      )}

      {insights.anomalies && insights.anomalies.isAnomaly && (
        <div className="insight-card alert">
          <h3>⚠️ Anomaly Detected</h3>
          <p>{insights.anomalies.details}</p>
        </div>
      )}
    </div>
  );
};

export default AIInsights;
```

## Backend (Node.js) Examples

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const bcrypt = require('bcryptjs');

exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.createUser = async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;

    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({
      name,
      email,
      password: hashedPassword,
      role,
      department,
      createdBy: req.user.id
    });

    await user.save();
    res.status(201).json({ message: 'User created successfully', user: user.toObject({ getters: true, versionKey: false, transform: (doc, ret) => { delete ret.password; return ret; } }) });
  } catch (error) {
    res.status(500).json({ message: 'Failed to create user', error: error.message });
  }
};

exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;

    if (updates.password) {
      updates.password = await bcrypt.hash(updates.password, 10);
    }

    const user = await User.findByIdAndUpdate(id, updates, { new: true }).select('-password');
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json({ message: 'User updated successfully', user });
  } catch (error) {
    res.status(500).json({ message: 'Failed to update user', error: error.message });
  }
};

exports.deleteUser = async (req, res) => {
  try {
    const { id } = req.params;
    const user = await User.findByIdAndDelete(id);
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ message: 'Failed to delete user', error: error.message });
  }
};
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const JWT_SECRET = process.env.JWT_SECRET;

exports.authenticate = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token provided' });
    }

    const decoded = jwt.verify(token, JWT_SECRET);
    const user = await User.findById(decoded.id).select('-password');

    if (!user) {
      return res.status(401).json({ message: 'User not found' });
    }

    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid or expired token' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: 'Access denied' });
    }
    next();
  };
};
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
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
  assignedTo: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'User', 
    required: true 
  },
  createdBy: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'User' 
  },
  dueDate: { type: Date },
  estimatedHours: { type: Number },
  actualHours: { type: Number, default: 0 },
  timeTracking: [{
    startTime: Date,
    endTime: Date,
    duration: Number
  }],
  tags: [String],
  attachments: [String]
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

## ML Service (FastAPI) Examples

### Main FastAPI Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from services.risk_predictor import RiskPredictor
from services.anomaly_detector import AnomalyDetector
from services.burnout_analyzer import BurnoutAnalyzer
from services.ticket_classifier import TicketClassifier

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize ML models
risk_predictor = RiskPredictor()
anomaly_detector = AnomalyDetector()
burnout_analyzer = BurnoutAnalyzer()
ticket_classifier = TicketClassifier()

class RiskPredictionRequest(BaseModel):
    userId: str
    taskCompletionRate: float
    averageTaskDelay: float
    missedDeadlines: int
    workHours: float

class RiskPredictionResponse(BaseModel):
    riskScore: float
    factors: List[str]
    recommendations: List[str]

@app.post("/api/ml/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    try:
        features = np.array([[
            request.taskCompletionRate,
            request.averageTaskDelay,
            request.missedDeadlines,
            request.workHours
        ]])
        
        risk_score = risk_predictor.predict(features)
        factors = risk_predictor.get_risk_factors(request.dict())
        recommendations = risk_predictor.get_recommendations(risk_score, factors)
        
        return RiskPredictionResponse(
            riskScore=float(risk_score),
            factors=factors,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    weeklyWorkHours: float
    tasksAssigned: int
    tasksCompleted: int
    overtimeHours: float
    consecutiveDaysWorked: int

class BurnoutAnalysisResponse(BaseModel):
    burnoutScore: float
    factors: List[str]
    suggestions: List[str]

@app.post("/api/ml/burnout-analysis", response_model=BurnoutAnalysisResponse)
async def analyze_burnout(request: BurnoutAnalysisRequest):
    try:
        score = burnout_analyzer.calculate_burnout_score(request.dict())
        factors = burnout_analyzer.identify_factors(request.dict())
        suggestions = burnout_analyzer.get_suggestions(score, factors)
        
        return BurnoutAnalysisResponse(
            burnoutScore=score,
            factors=factors,
            suggestions=suggestions
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    priority: str
    assignedTo: Optional[str]
    confidence: float

@app.post("/api/ml/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    try:
        result = ticket_classifier.classify(request.title, request.description)
        return TicketClassificationResponse(**result)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

### Burnout Analyzer Service

```python
# ml-service/services/burnout_analyzer.py
import numpy as np
from typing import List, Dict

class BurnoutAnalyzer:
    def __init__(self):
        self.thresholds = {
            'work_hours': 45,
            'overtime': 10,
            'consecutive_days': 6,
            'completion_rate': 0.7
        }
    
    def calculate_burnout_score(self, data: Dict) -> float:
        """Calculate burnout score (0-100)"""
        score = 0
        
        # Work hours factor (0-30 points)
        if data['weeklyWorkHours'] > self.thresholds['work_hours']:
            score += min(30, (data['weeklyWorkHours'] - self.thresholds['work_hours']) * 2)
        
        # Overtime factor (0-25 points)
        if data['overtimeHours'] > self.thresholds['overtime']:
            score += min(25, (data['overtimeHours'] - self.thresholds['overtime']) * 2.5)
        
        # Consecutive days factor (0-20 points)
        if data['consecutiveDaysWorked'] > self.thresholds['consecutive_days']:
            score += min(20, (data['consecutiveDaysWorked'] - self.thresholds['consecutive_days']) * 5)
        
        # Task completion rate factor (0-25 points)
        completion_rate = data['tasksCompleted'] / max(data['tasksAssigned'], 1)
        if completion_rate < self.thresholds['completion_rate']:
            score += (self.thresholds['completion_rate'] - completion_rate) * 100
        
        return min(100, score)
    
    def identify_factors(self, data: Dict) -> List[str]:
        """Identify contributing factors to burnout"""
        factors = []
        
        if data['weeklyWorkHours'] > self.thresholds['work_hours']:
            factors.append(f"Excessive work hours: {data['weeklyWorkHours']} hours/week")
        
        if data['overtimeHours'] > self.thresholds['overtime']:
            factors.append(f"High overtime: {data['overtimeHours']} hours")
        
        if data['consecutiveDaysWorked'] > self.thresholds['consecutive_days']:
            factors.append(f"No rest: {data['consecutiveDaysWorked']} consecutive days")
        
        completion_rate = data['tasksCompleted'] / max(data['tasksAssigned'], 1)
        if completion_rate < self.thresholds['completion_rate']:
            factors.append(f"Low completion rate: {completion_rate:.1%}")
        
        return factors
    
    def get_suggestions(self, score: float, factors: List[str]) -> List[str]:
        """Generate personalized suggestions"""
        suggestions = []
        
        if score >= 75:
            suggestions.append("Critical: Immediate workload reduction required")
            suggestions.append("Schedule mandatory time off")
        elif score >= 50:
            suggestions.append("High risk: Redistribute tasks to team members")
            suggestions.append("Ensure regular breaks and time off")
        elif score >= 25:
            suggestions.append("Moderate risk: Monitor workload closely")
            suggestions.append("Encourage work-life balance practices")
        
        if "overtime" in str(factors).lower():
            suggestions.append("Limit overtime to essential tasks only")
        
        if "consecutive days" in str(factors).lower():
            suggestions.append("Enforce weekly rest days")
        
        if "completion rate" in str(factors).lower():
            suggestions.append("Review task complexity and deadlines")
            suggestions.append("Provide additional support or training")
        
        return suggestions
```

### Risk Predictor Service

```python
# ml-service/services/risk_predictor.py
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import joblib
import os
from typing import List, Dict

class RiskPredictor:
    def __init__(self):
        model_path = os.getenv('MODEL_PATH', './models')
        self.model = None
        try:
            self.model = joblib.load(f"{model_path}/risk_model.pkl")
        except:
            # Initialize with default model if not found
            self.model = RandomForestClassifier(n_estimators=100, random_state=42)
            self._train_default_model()
    
    def _train_default_model(self):
        """Train with synthetic data for demo purposes"""
        X = np.random.rand(100, 4)
        y = (X[:, 0] < 0.5).astype(int)  # Simple rule
        self.model.fit(X, y)
    
    def predict(self, features: np.ndarray) -> float:
        """Predict risk score (0-100)"""
        if hasattr(self.model, 'predict_proba'):
            risk_prob = self.model.predict_proba(features)[0][1]
        else:
            risk_prob = self.model.predict(features)[0]
        return float(risk_prob * 100)
    
    def get_risk_factors(self, data: Dict) -> List[str]:
        """Identify risk factors from user data"""
        factors = []
        
        if data['taskCompletionRate'] < 0.7:
            factors.append(f"Low task completion: {data['taskCompletionRate']:.1%}")
        
        if data['averageTaskDelay'] > 3:
            factors.append(f"Frequent delays: {data['averageTaskDelay']} days average")
        
        if data['missedDeadlines'] > 5:
            factors.append(f"Missed deadlines: {data['missedDeadlines']} times")
        
        if data['workHours'] > 50:
            factors.append(f"Overwork: {data['workHours']} hours/week")
        
        return factors
    
    def get_recommendations(self, risk_score: float, factors: List[str]) -> List[str]:
        """Generate recommendations based on risk level"""
        recommendations = []
        
        if risk_score >= 75:
            recommendations.append("High risk: Immediate intervention required")
            recommendations.append("Reduce task load by 30-40%")
            recommendations.append("Schedule one-on-one meeting with manager")
        elif risk_score >= 50:
            recommendations.append("Medium risk: Monitor closely and provide support")
            recommendations.append("Review current task assignments")
        else:
            recommendations.append("Low risk: Continue regular monitoring")
        
        if "completion" in str(factors).lower():
            recommendations.append("Provide task prioritization training")
        
        if "delay" in str(factors).lower():
            recommendations.append("Identify and remove blockers")
        
        return recommendations
```

## Common Patterns

### Protected Route Pattern

```javascript
// frontend/src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, requiredRole }) => {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  if (!user) return <Navigate to="/login" />;
  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Time Tracking Pattern

```javascript
// frontend/src/components/TimeTracker.jsx
import { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [isTracking, setIsTracking] = useState(false);
  const [startTime, setStartTime] = useState(null);
  const [elapsed, setElapsed] = useState(0);

  useEffect(() => {
    let interval;
    if (isTracking) {
      interval = setInterval(() => {
        setElapsed(Date.now() - startTime);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isTracking, startTime]);

  const startTracking = () => {
    setStartTime(Date.now());
    setIsTracking(true);
  };

  const stopTracking = async () => {
    const endTime = Date.now();
    const duration = (endTime - startTime) / 1000 / 60; // minutes

    await axios.post(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`, {
      startTime: new Date(startTime),
      endTime: new Date(endTime),
      duration
    });

    setIsTracking(false);
    setElapsed(0);
  };

  const formatTime = (ms) => {
    const seconds = Math.floor(ms / 1000) % 60;
    const minutes = Math.floor(ms / 60000) % 60;
    const hours = Math.floor(ms / 3600000);
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer">{formatTime(elapsed)}</div>
      <button onClick={isTracking ? stopTracking : startTracking}>
        {isTracking ? 'Stop' : 'Start'}
      </button>
    </div>
  );
};

export default TimeTracker;
```

## Configuration

### MongoDB Connection

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`
