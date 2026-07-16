---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "show me how to implement JWT authentication for this system"
  - "how can I use the ML service for risk prediction"
  - "help me create user dashboards with task tracking"
  - "guide me through the ticket classification system"
  - "how do I set up the Kanban board feature"
  - "show me how to implement anomaly detection"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill provides expertise in using and extending the Enterprise User Management System with AI Analytics - a full-stack JavaScript application that combines user/task management with machine learning capabilities for risk detection, anomaly detection, burnout analysis, and predictive project insights.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Admin analytics and user performance insights

**Architecture**: React frontend + Node.js backend + FastAPI ML service + MongoDB

## Installation & Setup

### Prerequisites
```bash
# Required software
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
```

### Complete Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env
# Configure environment variables in .env

# ML Service setup
cd ../ml-service
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt

# Frontend setup
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env)**:
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=24h
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env)**:
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

**Frontend (.env)**:
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the System

```bash
# Terminal 1: Start MongoDB
mongod --dbpath /path/to/data

# Terminal 2: Start Backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 3: Start ML Service
cd ml-service
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000

# Terminal 4: Start Frontend
cd frontend
npm start
# Runs at http://localhost:3000
```

## Key API Endpoints

### Authentication APIs

```javascript
// Register new user
POST /api/auth/register
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
}

// Login
POST /api/auth/login
Content-Type: application/json

{
  "email": "john@example.com",
  "password": "securePassword123"
}

// Response includes JWT token
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "user_id",
    "name": "John Doe",
    "email": "john@example.com",
    "role": "user"
  }
}
```

### User Management APIs

```javascript
// Get all users (Admin only)
GET /api/users
Authorization: Bearer {token}

// Get user by ID
GET /api/users/:id
Authorization: Bearer {token}

// Update user
PUT /api/users/:id
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "John Updated",
  "role": "admin"
}

// Delete user (Admin only)
DELETE /api/users/:id
Authorization: Bearer {token}
```

### Task Management APIs

```javascript
// Create task
POST /api/tasks
Authorization: Bearer {token}
Content-Type: application/json

{
  "title": "Implement new feature",
  "description": "Add AI analytics dashboard",
  "assignedTo": "user_id",
  "status": "todo",
  "priority": "high",
  "dueDate": "2024-12-31"
}

// Update task status
PATCH /api/tasks/:id/status
Authorization: Bearer {token}
Content-Type: application/json

{
  "status": "in-progress"
}

// Get user tasks
GET /api/tasks/user/:userId
Authorization: Bearer {token}
```

### Support Ticket APIs

```javascript
// Create ticket
POST /api/tickets
Authorization: Bearer {token}
Content-Type: application/json

{
  "title": "Login issue",
  "description": "Cannot access dashboard after password reset",
  "priority": "high",
  "category": "technical"
}

// AI classification
POST /api/tickets/:id/classify
Authorization: Bearer {token}

// Response
{
  "category": "authentication",
  "priority": "high",
  "suggestedAssignee": "support_team_id",
  "confidence": 0.92
}
```

### ML Service APIs

```javascript
// Risk prediction
POST http://localhost:8000/api/ml/predict-risk
Content-Type: application/json

{
  "userId": "user_id",
  "features": {
    "loginFrequency": 5,
    "failedAttempts": 2,
    "locationChanges": 1,
    "timeOfAccess": 23,
    "dataAccessVolume": 150
  }
}

// Response
{
  "riskScore": 0.75,
  "riskLevel": "high",
  "factors": ["unusual_access_time", "high_failed_attempts"]
}

// Burnout detection
POST http://localhost:8000/api/ml/detect-burnout
Content-Type: application/json

{
  "userId": "user_id",
  "workload": {
    "tasksCount": 25,
    "hoursWorked": 55,
    "overtimeHours": 15,
    "missedDeadlines": 3,
    "weeksSinceBreak": 8
  }
}

// Anomaly detection
POST http://localhost:8000/api/ml/detect-anomaly
Content-Type: application/json

{
  "userId": "user_id",
  "activity": {
    "loginTime": "03:00",
    "location": "New York",
    "device": "Unknown Device",
    "activityPattern": [1, 0, 1, 1, 0]
  }
}
```

## Frontend Integration Patterns

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  const login = async (email, password) => {
    try {
      const response = await axios.post(`${API_URL}/api/auth/login`, {
        email,
        password
      });
      
      const { token, user } = response.data;
      localStorage.setItem('token', token);
      setToken(token);
      setUser(user);
      return { success: true };
    } catch (error) {
      return { success: false, error: error.response?.data?.message };
    }
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  useEffect(() => {
    if (token) {
      // Verify token and get user info
      axios.get(`${API_URL}/api/auth/me`, {
        headers: { Authorization: `Bearer ${token}` }
      })
      .then(res => setUser(res.data.user))
      .catch(() => logout());
    }
  }, [token]);

  return { user, token, login, logout };
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const API_URL = process.env.REACT_APP_API_URL;

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(
        `${API_URL}/api/tasks/user/${userId}`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
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

  const moveTask = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error moving task:', error);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = (e, status) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    moveTask(taskId, status);
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
          <h3>{status.toUpperCase()}</h3>
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
// src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const AIAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutLevel: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Fetch risk score
      const riskResponse = await axios.post(
        `${ML_API_URL}/api/ml/predict-risk`,
        { userId }
      );

      // Fetch burnout analysis
      const burnoutResponse = await axios.post(
        `${ML_API_URL}/api/ml/detect-burnout`,
        { userId }
      );

      // Fetch recent anomalies
      const anomalyResponse = await axios.get(
        `${ML_API_URL}/api/ml/anomalies/${userId}`
      );

      setAnalytics({
        riskScore: riskResponse.data,
        burnoutLevel: burnoutResponse.data,
        anomalies: anomalyResponse.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="ai-analytics-dashboard">
      <div className="analytics-card">
        <h3>Risk Score</h3>
        <div className={`score ${analytics.riskScore?.riskLevel}`}>
          {analytics.riskScore?.riskScore?.toFixed(2) || 'N/A'}
        </div>
        <p>Level: {analytics.riskScore?.riskLevel || 'Unknown'}</p>
      </div>

      <div className="analytics-card">
        <h3>Burnout Analysis</h3>
        <div className="burnout-indicator">
          {analytics.burnoutLevel?.score || 0}%
        </div>
        <ul>
          {analytics.burnoutLevel?.factors?.map((factor, idx) => (
            <li key={idx}>{factor}</li>
          ))}
        </ul>
      </div>

      <div className="analytics-card">
        <h3>Recent Anomalies</h3>
        <ul>
          {analytics.anomalies.map((anomaly, idx) => (
            <li key={idx}>
              <strong>{anomaly.type}</strong>: {anomaly.description}
              <span className="timestamp">{anomaly.timestamp}</span>
            </li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Backend Implementation Patterns

### JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      throw new Error('No token provided');
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id);

    if (!user) {
      throw new Error('User not found');
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Please authenticate' });
  }
};

const adminAuth = async (req, res, next) => {
  try {
    await auth(req, res, () => {
      if (req.user.role !== 'admin') {
        return res.status(403).json({ error: 'Admin access required' });
      }
      next();
    });
  } catch (error) {
    res.status(403).json({ error: 'Admin access required' });
  }
};

module.exports = { auth, adminAuth };
```

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

// Register user
exports.register = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;

    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' });
    }

    // Hash password
    const hashedPassword = await bcrypt.hash(password, 10);

    // Create user
    const user = new User({
      name,
      email,
      password: hashedPassword,
      role: role || 'user'
    });

    await user.save();

    // Generate token
    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );

    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Login user
exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;

    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRE }
    );

    res.json({
      success: true,
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Get all users
exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;

    // Don't allow password updates through this endpoint
    delete updates.password;

    const user = await User.findByIdAndUpdate(
      id,
      updates,
      { new: true, runValidators: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL;

// Create task
exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id
    });

    await task.save();

    // Predict task completion using ML service
    try {
      const prediction = await axios.post(
        `${ML_SERVICE_URL}/api/ml/predict-completion`,
        {
          taskId: task._id,
          priority: task.priority,
          assignedTo: task.assignedTo,
          dueDate: task.dueDate
        }
      );
      task.estimatedCompletion = prediction.data.estimatedDate;
      await task.save();
    } catch (mlError) {
      console.error('ML prediction failed:', mlError);
    }

    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Get user tasks
exports.getUserTasks = async (req, res) => {
  try {
    const { userId } = req.params;
    const tasks = await Task.find({ assignedTo: userId })
      .populate('createdBy', 'name email')
      .sort({ createdAt: -1 });

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// Update task status
exports.updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;

    const task = await Task.findByIdAndUpdate(
      id,
      { status, updatedAt: Date.now() },
      { new: true }
    );

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    res.json(task);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```

## ML Service Implementation

### Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
from river import ensemble, tree
import numpy as np
import joblib
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
        """Load pre-trained model or create new one"""
        if os.path.exists(self.model_path):
            self.model = joblib.load(self.model_path)
        else:
            self.model = RandomForestClassifier(
                n_estimators=100,
                max_depth=10,
                random_state=42
            )
    
    def predict(self, features):
        """
        Predict risk score
        
        Args:
            features (dict): User activity features
                - loginFrequency: int
                - failedAttempts: int
                - locationChanges: int
                - timeOfAccess: int (0-23)
                - dataAccessVolume: int
        
        Returns:
            dict: Risk score and level
        """
        feature_vector = np.array([
            features.get('loginFrequency', 0),
            features.get('failedAttempts', 0),
            features.get('locationChanges', 0),
            features.get('timeOfAccess', 12),
            features.get('dataAccessVolume', 0)
        ]).reshape(1, -1)
        
        # Get probability from the model
        if hasattr(self.model, 'predict_proba'):
            risk_score = self.model.predict_proba(feature_vector)[0][1]
        else:
            risk_score = self.model.predict(feature_vector)[0]
        
        # Determine risk level
        if risk_score < 0.3:
            risk_level = 'low'
        elif risk_score < 0.7:
            risk_level = 'medium'
        else:
            risk_level = 'high'
        
        # Identify risk factors
        factors = []
        if features.get('failedAttempts', 0) > 2:
            factors.append('high_failed_attempts')
        if features.get('timeOfAccess', 12) < 6 or features.get('timeOfAccess', 12) > 22:
            factors.append('unusual_access_time')
        if features.get('locationChanges', 0) > 2:
            factors.append('multiple_location_changes')
        if features.get('dataAccessVolume', 0) > 100:
            factors.append('high_data_access')
        
        return {
            'riskScore': float(risk_score),
            'riskLevel': risk_level,
            'factors': factors
        }
    
    def train_online(self, features, label):
        """Update model with new data point"""
        feature_dict = {
            'loginFrequency': features.get('loginFrequency', 0),
            'failedAttempts': features.get('failedAttempts', 0),
            'locationChanges': features.get('locationChanges', 0),
            'timeOfAccess': features.get('timeOfAccess', 12),
            'dataAccessVolume': features.get('dataAccessVolume', 0)
        }
        self.online_model.learn_one(feature_dict, label)
```

### Burnout Detection

```python
# ml-service/models/burnout_detector.py
import numpy as np

class BurnoutDetector:
    def __init__(self):
        # Thresholds for burnout indicators
        self.thresholds = {
            'tasks_count': 20,
            'hours_worked': 50,
            'overtime_hours': 10,
            'missed_deadlines': 2,
            'weeks_since_break': 6
        }
        self.weights = {
            'tasks_count': 0.2,
            'hours_worked': 0.25,
            'overtime_hours': 0.25,
            'missed_deadlines': 0.15,
            'weeks_since_break': 0.15
        }
    
    def detect(self, workload):
        """
        Detect burnout risk
        
        Args:
            workload (dict):
                - tasksCount: int
                - hoursWorked: int
                - overtimeHours: int
                - missedDeadlines: int
                - weeksSinceBreak: int
        
        Returns:
            dict: Burnout score and factors
        """
        score = 0
        factors = []
        
        # Calculate normalized scores for each factor
        for key, threshold in self.thresholds.items():
            camel_key = self._to_camel_case(key)
            value = workload.get(camel_key, 0)
            normalized = min(value / threshold, 1.0)
            score += normalized * self.weights[key]
            
            if normalized > 0.8:
                factors.append(key)
        
        burnout_score = int(score * 100)
        
        # Determine level
        if burnout_score < 30:
            level = 'low'
        elif burnout_score < 60:
            level = 'moderate'
        else:
            level = 'high'
        
        recommendations = self._get_recommendations(factors)
        
        return {
            'score': burnout_score,
            'level': level,
            'factors': factors,
            'recommendations': recommendations
        }
    
    def _to_camel_case(self, snake_str):
        """Convert snake_case to camelCase"""
        components = snake_str.split('_')
        return components[0] + ''.join(x.title() for x in components[1:])
    
    def _get_recommendations(self, factors):
        """Generate recommendations based on factors"""
        recommendations = []
        
        if 'tasks_count' in factors:
            recommendations.append('Consider redistributing tasks')
        if 'hours_worked' in factors or 'overtime_hours' in factors:
            recommendations.append('Reduce working hours and overtime')
        if 'missed_deadlines' in factors:
            recommendations.append('Review task priorities and deadlines')
        if 'weeks_since_break' in factors:
            recommendations.append('Schedule time off or vacation')
        
        return recommendations
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Dict, Optional
import uvicorn

from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector
from models.anomaly_detector import AnomalyDetector

app = FastAPI(title="Enterprise User Management ML Service")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize models
risk_predictor = RiskPredictor()
burnout_detector = BurnoutDetector()
anomaly_detector = AnomalyDetector()

# Request models
class RiskPredictionRequest(BaseModel):
    userId: str
    features: Dict[str, int]

class BurnoutDetectionRequest(BaseModel):
    userId: str
    workload: Dict[str, int]

class AnomalyDetectionRequest(BaseModel):
    userId: str
    activity: Dict[str, any]

# Endpoints
@app.get("/")
def read_root():
    return {"message": "Enterprise User Management ML Service"}

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score"""
    try:
        result = risk_predictor.predict(request.features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    """Detect employee burnout risk"""
    try:
        result = burnout_detector.detect(request.workload)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user activity"""
    try:
        result = anomaly_detector.detect(
            request.userId,
            request.activity
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/ml/anomalies/{user_id}")
async def get_user_anomalies(user_id: str, limit: int = 10):
    """Get recent anomalies for a user"""
    try:
        anomalies = anomaly_detector.get_recent_anomalies(user_id, limit)
        return anomalies
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0
