---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user behavior"
  - "create user management dashboard with ML insights"
  - "configure burnout detection and risk prediction"
  - "build task management with AI ticket routing"
  - "integrate AI anomaly detection in user system"
  - "deploy enterprise user management with FastAPI ML"
  - "implement JWT authentication with AI analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What It Does

Enterprise User Management System is a full-stack application combining user/task management with AI-powered analytics. It provides:

- **User Management**: Role-based access control, authentication (JWT), user CRUD operations
- **Task Management**: Kanban boards, time tracking, assignment workflows
- **Support Tickets**: AI-based classification and automatic routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Centralized monitoring, audit logs, alerts

The system uses a React frontend, Node.js/Express backend, MongoDB database, and FastAPI ML service with scikit-learn and River for online learning.

## Installation

### Prerequisites

```bash
# Required: Node.js 14+, Python 3.8+, MongoDB
node --version
python --version
mongod --version
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
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

## Key API Endpoints

### Authentication (Backend)

```javascript
// POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePass123",
  "role": "user" // or "admin"
}

// POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePass123"
}

// Response includes JWT token
{
  "token": "eyJhbGc...",
  "user": { "id": "...", "name": "John Doe", "role": "user" }
}
```

### User Management (Backend)

```javascript
// GET /api/users - Get all users (admin only)
// GET /api/users/:id - Get user by ID
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user (admin only)

// Headers required for authenticated routes:
{
  "Authorization": "Bearer <JWT_TOKEN>"
}
```

### Task Management (Backend)

```javascript
// POST /api/tasks - Create task
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth",
  "assignedTo": "user_id",
  "priority": "high", // low, medium, high
  "status": "todo", // todo, in-progress, done
  "dueDate": "2026-05-01T00:00:00Z"
}

// GET /api/tasks - Get all tasks
// GET /api/tasks/user/:userId - Get tasks for user
// PUT /api/tasks/:id - Update task
// PATCH /api/tasks/:id/status - Update task status
{
  "status": "in-progress"
}
```

### Support Tickets (Backend)

```javascript
// POST /api/tickets - Create support ticket
{
  "title": "Login issue",
  "description": "Cannot log in with correct credentials",
  "priority": "high",
  "category": "technical" // technical, billing, general
}

// GET /api/tickets - Get all tickets (admin)
// GET /api/tickets/user/:userId - Get user's tickets
// PUT /api/tickets/:id - Update ticket
// POST /api/tickets/:id/classify - AI classification
```

### AI Analytics (ML Service)

```python
# POST /api/ml/risk-prediction
{
  "userId": "user_id",
  "loginAttempts": 5,
  "failedLogins": 3,
  "lastLogin": "2026-04-15T10:30:00Z",
  "taskCompletionRate": 0.65,
  "avgResponseTime": 120
}

# POST /api/ml/anomaly-detection
{
  "userId": "user_id",
  "behaviorMetrics": {
    "loginTime": "03:00:00",
    "loginLocation": "unknown",
    "accessPatterns": ["unusual_endpoint"],
    "dataAccess": 500
  }
}

# POST /api/ml/burnout-detection
{
  "userId": "user_id",
  "workloadMetrics": {
    "tasksAssigned": 25,
    "tasksCompleted": 10,
    "avgWorkHours": 12,
    "weekendWork": true,
    "overtimeHours": 20
  }
}

# POST /api/ml/predict-project-delay
{
  "projectId": "project_id",
  "tasksTotal": 50,
  "tasksCompleted": 15,
  "daysElapsed": 30,
  "daysRemaining": 60,
  "teamSize": 5,
  "avgVelocity": 2.5
}

# POST /api/ml/classify-ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when trying to view admin panel"
}
```

## Configuration

### Backend Configuration (backend/config/config.js)

```javascript
module.exports = {
  port: process.env.PORT || 5000,
  mongoUri: process.env.MONGODB_URI,
  jwtSecret: process.env.JWT_SECRET,
  jwtExpire: process.env.JWT_EXPIRE || '7d',
  mlServiceUrl: process.env.ML_SERVICE_URL || 'http://localhost:8000',
  corsOrigins: process.env.CORS_ORIGINS?.split(',') || ['http://localhost:3000'],
  rateLimit: {
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100 // limit each IP to 100 requests per windowMs
  }
};
```

### ML Service Configuration (ml-service/config.py)

```python
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    mongodb_uri: str = os.getenv("MONGODB_URI", "mongodb://localhost:27017/enterprise-user-mgmt")
    model_path: str = os.getenv("MODEL_PATH", "./models")
    log_level: str = os.getenv("LOG_LEVEL", "INFO")
    
    # Model hyperparameters
    risk_threshold: float = 0.7
    anomaly_threshold: float = 0.8
    burnout_threshold: float = 0.75
    
    class Config:
        env_file = ".env"

settings = Settings()
```

## Code Examples

### Backend: User Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization?.startsWith('Bearer')) {
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

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        message: `User role ${req.user.role} is not authorized to access this route` 
      });
    }
    next();
  };
};

module.exports = { protect, authorize };
```

### Backend: Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

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
    
    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { status } = req.body;
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ success: false, error: 'Task not found' });
    }
    
    task.status = status;
    if (status === 'done') {
      task.completedAt = new Date();
    }
    
    await task.save();
    res.status(200).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name')
      .sort('-createdAt');
    
    res.status(200).json({ success: true, count: tasks.length, data: tasks });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};
```

### ML Service: Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
import numpy as np
import joblib
import os

class RiskPredictor:
    def __init__(self, model_path='./models'):
        self.model_path = model_path
        self.model = None
        self.scaler = StandardScaler()
        self.load_or_initialize()
    
    def load_or_initialize(self):
        model_file = os.path.join(self.model_path, 'risk_model.pkl')
        scaler_file = os.path.join(self.model_path, 'risk_scaler.pkl')
        
        if os.path.exists(model_file):
            self.model = joblib.load(model_file)
            self.scaler = joblib.load(scaler_file)
        else:
            self.model = RandomForestClassifier(
                n_estimators=100,
                max_depth=10,
                random_state=42
            )
    
    def extract_features(self, user_data):
        """Extract features from user behavior data"""
        features = [
            user_data.get('loginAttempts', 0),
            user_data.get('failedLogins', 0),
            user_data.get('taskCompletionRate', 0),
            user_data.get('avgResponseTime', 0),
            user_data.get('suspiciousActivities', 0),
            user_data.get('dataAccessVolume', 0)
        ]
        return np.array(features).reshape(1, -1)
    
    def predict(self, user_data):
        """Predict risk level for a user"""
        features = self.extract_features(user_data)
        features_scaled = self.scaler.transform(features)
        
        if hasattr(self.model, 'predict_proba'):
            risk_score = self.model.predict_proba(features_scaled)[0][1]
        else:
            risk_score = float(self.model.predict(features_scaled)[0])
        
        return {
            'userId': user_data.get('userId'),
            'riskScore': round(risk_score, 3),
            'riskLevel': self._categorize_risk(risk_score),
            'factors': self._identify_risk_factors(user_data, features[0])
        }
    
    def _categorize_risk(self, score):
        if score >= 0.7:
            return 'high'
        elif score >= 0.4:
            return 'medium'
        return 'low'
    
    def _identify_risk_factors(self, user_data, features):
        factors = []
        if user_data.get('failedLogins', 0) > 3:
            factors.append('Multiple failed login attempts')
        if user_data.get('taskCompletionRate', 1) < 0.5:
            factors.append('Low task completion rate')
        if user_data.get('avgResponseTime', 0) > 300:
            factors.append('Slow response time')
        return factors
    
    def save(self):
        os.makedirs(self.model_path, exist_ok=True)
        joblib.dump(self.model, os.path.join(self.model_path, 'risk_model.pkl'))
        joblib.dump(self.scaler, os.path.join(self.model_path, 'risk_scaler.pkl'))
```

### ML Service: Burnout Detection

```python
# ml-service/models/burnout_detector.py
from river import anomaly
from river import preprocessing
import numpy as np

class BurnoutDetector:
    def __init__(self):
        self.model = anomaly.HalfSpaceTrees(
            n_trees=25,
            height=8,
            window_size=250,
            seed=42
        )
        self.scaler = preprocessing.StandardScaler()
    
    def extract_features(self, workload_data):
        """Extract burnout-related features"""
        return {
            'tasks_ratio': workload_data.get('tasksCompleted', 0) / max(workload_data.get('tasksAssigned', 1), 1),
            'avg_work_hours': workload_data.get('avgWorkHours', 8),
            'weekend_work': 1 if workload_data.get('weekendWork', False) else 0,
            'overtime_hours': workload_data.get('overtimeHours', 0),
            'stress_level': workload_data.get('stressLevel', 0),
            'break_time': workload_data.get('breakTime', 60)
        }
    
    def detect(self, workload_data):
        """Detect burnout risk using anomaly detection"""
        features = self.extract_features(workload_data)
        
        # Scale features
        scaled_features = {}
        for key, value in features.items():
            self.scaler.learn_one({key: value})
            scaled_features[key] = self.scaler.transform_one({key: value})[key]
        
        # Get anomaly score
        score = self.model.score_one(scaled_features)
        self.model.learn_one(scaled_features)
        
        burnout_score = min(score / 1.5, 1.0)  # Normalize
        
        return {
            'userId': workload_data.get('userId'),
            'burnoutScore': round(burnout_score, 3),
            'burnoutLevel': self._categorize_burnout(burnout_score),
            'warnings': self._generate_warnings(features),
            'recommendations': self._generate_recommendations(features)
        }
    
    def _categorize_burnout(self, score):
        if score >= 0.75:
            return 'critical'
        elif score >= 0.5:
            return 'high'
        elif score >= 0.3:
            return 'moderate'
        return 'low'
    
    def _generate_warnings(self, features):
        warnings = []
        if features['avg_work_hours'] > 10:
            warnings.append('Excessive working hours detected')
        if features['weekend_work'] == 1:
            warnings.append('Working on weekends')
        if features['overtime_hours'] > 15:
            warnings.append('High overtime hours')
        if features['tasks_ratio'] < 0.5:
            warnings.append('Low task completion rate - possible overload')
        return warnings
    
    def _generate_recommendations(self, features):
        recommendations = []
        if features['avg_work_hours'] > 10:
            recommendations.append('Reduce daily work hours to 8-9 hours')
        if features['weekend_work'] == 1:
            recommendations.append('Take weekends off for better work-life balance')
        if features['break_time'] < 60:
            recommendations.append('Take regular breaks throughout the day')
        if features['tasks_ratio'] < 0.5:
            recommendations.append('Reassess task allocation and priorities')
        return recommendations
```

### ML Service: FastAPI Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Dict, List
from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector
from models.ticket_classifier import TicketClassifier
import logging

app = FastAPI(title="Enterprise User Management ML Service")
logging.basicConfig(level=logging.INFO)

# Initialize models
risk_predictor = RiskPredictor()
burnout_detector = BurnoutDetector()
ticket_classifier = TicketClassifier()

class RiskPredictionRequest(BaseModel):
    userId: str
    loginAttempts: int = 0
    failedLogins: int = 0
    taskCompletionRate: float = 1.0
    avgResponseTime: int = 0
    suspiciousActivities: int = 0
    dataAccessVolume: int = 0

class BurnoutDetectionRequest(BaseModel):
    userId: str
    tasksAssigned: int
    tasksCompleted: int
    avgWorkHours: float
    weekendWork: bool = False
    overtimeHours: int = 0
    stressLevel: int = 0
    breakTime: int = 60

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

@app.post("/api/ml/risk-prediction")
async def predict_risk(request: RiskPredictionRequest):
    try:
        result = risk_predictor.predict(request.dict())
        return {"success": True, "data": result}
    except Exception as e:
        logging.error(f"Risk prediction error: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        result = burnout_detector.detect(request.dict())
        return {"success": True, "data": result}
    except Exception as e:
        logging.error(f"Burnout detection error: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        result = ticket_classifier.classify(request.dict())
        return {"success": True, "data": result}
    except Exception as e:
        logging.error(f"Ticket classification error: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/ml/health")
async def health_check():
    return {
        "status": "healthy",
        "models": {
            "risk_predictor": "loaded",
            "burnout_detector": "loaded",
            "ticket_classifier": "loaded"
        }
    }
```

### Frontend: React Hook for AI Analytics

```javascript
// frontend/src/hooks/useAIAnalytics.js
import { useState } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL || 'http://localhost:8000';

export const useAIAnalytics = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const predictRisk = async (userData) => {
    setLoading(true);
    setError(null);
    try {
      const response = await axios.post(
        `${ML_API_URL}/api/ml/risk-prediction`,
        userData
      );
      return response.data.data;
    } catch (err) {
      setError(err.response?.data?.detail || 'Risk prediction failed');
      throw err;
    } finally {
      setLoading(false);
    }
  };

  const detectBurnout = async (workloadData) => {
    setLoading(true);
    setError(null);
    try {
      const response = await axios.post(
        `${ML_API_URL}/api/ml/burnout-detection`,
        workloadData
      );
      return response.data.data;
    } catch (err) {
      setError(err.response?.data?.detail || 'Burnout detection failed');
      throw err;
    } finally {
      setLoading(false);
    }
  };

  const classifyTicket = async (ticketData) => {
    setLoading(true);
    setError(null);
    try {
      const response = await axios.post(
        `${ML_API_URL}/api/ml/classify-ticket`,
        ticketData
      );
      return response.data.data;
    } catch (err) {
      setError(err.response?.data?.detail || 'Ticket classification failed');
      throw err;
    } finally {
      setLoading(false);
    }
  };

  return {
    predictRisk,
    detectBurnout,
    classifyTicket,
    loading,
    error
  };
};
```

### Frontend: Admin Dashboard Component

```javascript
// frontend/src/components/AdminDashboard.jsx
import React, { useEffect, useState } from 'react';
import { useAIAnalytics } from '../hooks/useAIAnalytics';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskAnalysis, setRiskAnalysis] = useState([]);
  const { predictRisk, loading } = useAIAnalytics();
  
  const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchUsers();
  }, []);

  const fetchUsers = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/users`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      setUsers(response.data.data);
      analyzeUserRisks(response.data.data);
    } catch (error) {
      console.error('Error fetching users:', error);
    }
  };

  const analyzeUserRisks = async (userList) => {
    const risks = [];
    for (const user of userList) {
      try {
        const riskData = await predictRisk({
          userId: user._id,
          loginAttempts: user.loginAttempts || 0,
          failedLogins: user.failedLogins || 0,
          taskCompletionRate: user.taskCompletionRate || 1.0,
          avgResponseTime: user.avgResponseTime || 0
        });
        risks.push({ ...user, risk: riskData });
      } catch (error) {
        console.error(`Risk analysis failed for user ${user._id}:`, error);
      }
    }
    setRiskAnalysis(risks);
  };

  const getRiskColor = (level) => {
    switch (level) {
      case 'high': return '#ff4444';
      case 'medium': return '#ffaa00';
      case 'low': return '#44ff44';
      default: return '#cccccc';
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="stats-grid">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p className="stat-number">{users.length}</p>
        </div>
        <div className="stat-card">
          <h3>High Risk Users</h3>
          <p className="stat-number">
            {riskAnalysis.filter(u => u.risk?.riskLevel === 'high').length}
          </p>
        </div>
      </div>

      <div className="risk-analysis-section">
        <h2>User Risk Analysis</h2>
        {loading ? (
          <p>Analyzing risks...</p>
        ) : (
          <table className="risk-table">
            <thead>
              <tr>
                <th>User</th>
                <th>Email</th>
                <th>Risk Score</th>
                <th>Risk Level</th>
                <th>Factors</th>
              </tr>
            </thead>
            <tbody>
              {riskAnalysis.map(user => (
                <tr key={user._id}>
                  <td>{user.name}</td>
                  <td>{user.email}</td>
                  <td>{user.risk?.riskScore || 'N/A'}</td>
                  <td>
                    <span 
                      className="risk-badge"
                      style={{ backgroundColor: getRiskColor(user.risk?.riskLevel) }}
                    >
                      {user.risk?.riskLevel || 'unknown'}
                    </span>
                  </td>
                  <td>
                    <ul className="risk-factors">
                      {user.risk?.factors?.map((factor, idx) => (
                        <li key={idx}>{factor}</li>
                      ))}
                    </ul>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        )}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Common Patterns

### Pattern 1: Scheduled AI Analysis

```javascript
// backend/jobs/aiAnalysisJob.js
const cron = require('node-cron');
const axios = require('axios');
const User = require('../models/User');

// Run AI analysis every day at midnight
const scheduleAIAnalysis = () => {
  cron.schedule('0 0 * * *', async () => {
    console.log('Running daily AI analysis...');
    
    const users = await User.find({ isActive: true });
    
    for (const user of users) {
      try {
        // Get user metrics
        const metrics = await getUserMetrics(user._id);
        
        // Risk prediction
        const riskResult = await axios.post(
          `${process.env.ML_SERVICE_URL}/api/ml/risk-prediction`,
          { userId: user._id, ...metrics }
        );
        
        // Burnout detection
        const burnoutResult = await axios.post(
          `${process.env.ML_SERVICE_URL}/api/ml/burnout-detection`,
          { userId: user._id, ...metrics }
        );
        
        // Save results
        await saveAnalysisResults(user._id, {
          risk: riskResult.data.data,
          burnout: burnoutResult.data.data
        });
        
        // Send alerts if needed
        if (riskResult.data.data.riskLevel === 'high') {
          await sendAlert(user, 'High risk detected');
        }
        
      } catch (error) {
        console.error(`Analysis failed for user ${user._id}:`, error);
      }
    }
  });
};

module.exports = { scheduleAIAnalysis };
```

### Pattern 2: Real-time Ticket Classification

```javascript
// backend/controllers/ticketController.js
const axios = require('axios');
const Ticket = require('../models/Ticket');

exports.createTicket = async (req, res) => {
  try {
    const { title, description, priority } = req.body;
    
    // Get AI classification
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/api/ml/classify-ticket`,
      { title, description }
    );
    
    const { category, department, suggestedPriority } = mlResponse.data.data;
    
    // Create ticket with AI insights
    const ticket = await Ticket.create({
      title,
      description,
      priority: priority || suggestedPriority,
      category,
      assignedDepartment: department,
      createdBy: req.user.id,
      aiClassified: true
    });
    
    // Auto-assign to department
    await autoAssignTicket(ticket, department);
    
    res.status(201).json({ success: true, data: ticket });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
};
```

### Pattern 3: Task Time Tracking

```javascript
// frontend/src/components/TaskTimer.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskTimer = ({ taskId, onComplete }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  
  const API_URL = process.env.REACT_APP_API_URL;
  const token = localStorage.getItem('token');

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
