---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and ticket automation
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "add AI ticket classification and routing"
  - "build admin panel with role-based access"
  - "integrate burnout detection and risk prediction"
  - "deploy user management system with ML service"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system that combines React frontend, Node.js backend, and FastAPI ML service to provide intelligent user administration, task management, and AI-driven insights including risk detection, anomaly detection, burnout analysis, and predictive analytics.

## What This Project Does

- **User Management**: Secure authentication, role-based access control, user CRUD operations
- **Task Management**: Kanban board interface, time tracking, task assignment and monitoring
- **Support System**: Ticket creation, tracking, and AI-powered classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout detection, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, user activity monitoring
- **Real-time Insights**: Performance metrics, workload analysis, predictive insights

## Installation

### Prerequisites
```bash
# Required
node >= 14.x
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
MONGODB_URI=mongodb://localhost:27017/enterprise_db
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
EOF

npm start
# Backend runs at http://localhost:5000
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

uvicorn main:app --reload --host 0.0.0.0 --port 8000
# ML service runs at http://localhost:8000
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

npm start
# Frontend runs at http://localhost:3000
```

## Key API Endpoints

### Authentication
```javascript
// POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepass123",
  "role": "user"
}

// POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securepass123"
}
// Returns: { token, user }
```

### User Management (Admin)
```javascript
// GET /api/users - List all users
// GET /api/users/:id - Get user details
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user
// POST /api/users - Create new user
```

### Task Management
```javascript
// GET /api/tasks - Get user tasks
// POST /api/tasks - Create task
{
  "title": "Implement authentication",
  "description": "Add JWT auth",
  "assignedTo": "userId",
  "status": "todo",
  "priority": "high",
  "dueDate": "2026-05-01"
}

// PUT /api/tasks/:id - Update task status
// DELETE /api/tasks/:id - Delete task
```

### Support Tickets
```javascript
// POST /api/tickets
{
  "subject": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// GET /api/tickets - Get all tickets
// PUT /api/tickets/:id - Update ticket
```

### AI Analytics Endpoints
```python
# POST /ml/classify-ticket
{
  "subject": "Password reset needed",
  "description": "Cannot remember password"
}
# Returns: { "category": "authentication", "priority": "medium" }

# POST /ml/detect-risk
{
  "userId": "12345",
  "loginAttempts": 5,
  "failedLogins": 3,
  "lastActivity": "2026-04-15T10:00:00Z"
}
# Returns: { "riskScore": 0.75, "riskLevel": "high" }

# POST /ml/predict-burnout
{
  "userId": "12345",
  "tasksCompleted": 45,
  "averageWorkHours": 11,
  "missedDeadlines": 3
}
# Returns: { "burnoutScore": 0.82, "recommendation": "reduce workload" }

# POST /ml/detect-anomaly
{
  "userId": "12345",
  "loginTime": "03:00",
  "loginLocation": "Unknown",
  "deviceFingerprint": "abc123"
}
# Returns: { "isAnomaly": true, "confidence": 0.89 }
```

## Real Code Examples

### Backend: User Authentication Middleware
```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && 
      req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ 
      error: 'Not authorized to access this route' 
    });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id);
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

const authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        error: 'User role not authorized' 
      });
    }
    next();
  };
};

module.exports = { protect, authorize };
```

### Backend: Task Controller
```javascript
// controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });

    // Get AI insights for task
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/ml/predict-delay`,
      {
        taskComplexity: req.body.complexity,
        assigneeWorkload: await getAssigneeWorkload(req.body.assignedTo),
        estimatedHours: req.body.estimatedHours
      }
    );

    task.delayPrediction = mlResponse.data;
    await task.save();

    res.status(201).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status, updatedAt: Date.now() },
      { new: true, runValidators: true }
    );

    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    res.status(200).json({ success: true, data: task });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

const getAssigneeWorkload = async (userId) => {
  const tasks = await Task.find({
    assignedTo: userId,
    status: { $in: ['todo', 'in-progress'] }
  });
  return tasks.length;
};
```

### Frontend: User Dashboard Component
```javascript
// src/components/UserDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [insights, setInsights] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const token = localStorage.getItem('token');
      const config = {
        headers: { Authorization: `Bearer ${token}` }
      };

      const [tasksRes, insightsRes] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/api/users/insights`, config)
      ]);

      setTasks(tasksRes.data.data);
      setInsights(insightsRes.data.data);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
    } finally {
      setLoading(false);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.put(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchDashboardData();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {insights?.burnoutScore > 0.7 && (
        <div className="alert alert-warning">
          Warning: High burnout risk detected. Consider reducing workload.
        </div>
      )}

      <div className="kanban-board">
        {['todo', 'in-progress', 'done'].map(status => (
          <div key={status} className="kanban-column">
            <h2>{status.toUpperCase()}</h2>
            {tasks
              .filter(task => task.status === status)
              .map(task => (
                <div key={task._id} className="task-card">
                  <h3>{task.title}</h3>
                  <p>{task.description}</p>
                  <div className="task-actions">
                    {status !== 'done' && (
                      <button onClick={() => moveTask(task._id, 
                        status === 'todo' ? 'in-progress' : 'done'
                      )}>
                        Move →
                      </button>
                    )}
                  </div>
                </div>
              ))}
          </div>
        ))}
      </div>
    </div>
  );
};

export default UserDashboard;
```

### ML Service: Risk Detection Model
```python
# ml-service/models/risk_detector.py
from river import linear_model, preprocessing
import numpy as np

class RiskDetector:
    def __init__(self):
        self.model = preprocessing.StandardScaler() | linear_model.LogisticRegression()
        self.is_trained = False
    
    def predict_risk(self, user_data):
        """
        Predict user risk score based on behavior
        
        Args:
            user_data: dict with keys:
                - login_attempts: int
                - failed_logins: int
                - unusual_activity: int
                - last_login_hours: float
        
        Returns:
            dict: {'riskScore': float, 'riskLevel': str}
        """
        features = {
            'login_attempts': user_data.get('loginAttempts', 0),
            'failed_logins': user_data.get('failedLogins', 0),
            'unusual_activity': user_data.get('unusualActivity', 0),
            'last_login_hours': user_data.get('lastLoginHours', 0)
        }
        
        # Calculate risk score
        risk_score = self._calculate_risk(features)
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = 'high'
        elif risk_score >= 0.4:
            risk_level = 'medium'
        else:
            risk_level = 'low'
        
        return {
            'riskScore': round(risk_score, 2),
            'riskLevel': risk_level,
            'factors': self._get_risk_factors(features)
        }
    
    def _calculate_risk(self, features):
        # Simple weighted scoring
        score = 0.0
        
        if features['failed_logins'] > 3:
            score += 0.3
        
        if features['login_attempts'] > 5:
            score += 0.2
        
        if features['unusual_activity'] > 0:
            score += 0.3
        
        if features['last_login_hours'] > 168:  # 1 week
            score += 0.2
        
        return min(score, 1.0)
    
    def _get_risk_factors(self, features):
        factors = []
        if features['failed_logins'] > 3:
            factors.append('Multiple failed login attempts')
        if features['unusual_activity'] > 0:
            factors.append('Unusual activity detected')
        if features['last_login_hours'] > 168:
            factors.append('Prolonged inactivity')
        return factors
```

### ML Service: FastAPI Main Application
```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import uvicorn
from models.risk_detector import RiskDetector
from models.burnout_predictor import BurnoutPredictor
from models.ticket_classifier import TicketClassifier

app = FastAPI(title="Enterprise User Management ML Service")

# Initialize models
risk_detector = RiskDetector()
burnout_predictor = BurnoutPredictor()
ticket_classifier = TicketClassifier()

class TicketData(BaseModel):
    subject: str
    description: str

class RiskData(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    unusualActivity: int = 0
    lastLoginHours: float = 0

class BurnoutData(BaseModel):
    userId: str
    tasksCompleted: int
    averageWorkHours: float
    missedDeadlines: int
    weeklyTasks: int

@app.get("/")
async def root():
    return {"message": "Enterprise ML Service API", "status": "active"}

@app.post("/ml/classify-ticket")
async def classify_ticket(data: TicketData):
    """Classify support ticket and assign priority"""
    try:
        result = ticket_classifier.classify(
            subject=data.subject,
            description=data.description
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/detect-risk")
async def detect_risk(data: RiskData):
    """Detect user risk based on behavior patterns"""
    try:
        result = risk_detector.predict_risk({
            'loginAttempts': data.loginAttempts,
            'failedLogins': data.failedLogins,
            'unusualActivity': data.unusualActivity,
            'lastLoginHours': data.lastLoginHours
        })
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/ml/predict-burnout")
async def predict_burnout(data: BurnoutData):
    """Predict employee burnout risk"""
    try:
        result = burnout_predictor.predict({
            'tasksCompleted': data.tasksCompleted,
            'averageWorkHours': data.averageWorkHours,
            'missedDeadlines': data.missedDeadlines,
            'weeklyTasks': data.weeklyTasks
        })
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "models_loaded": True}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Frontend: AI Insights Component
```javascript
// src/components/AIInsights.jsx
import React, { useEffect, useState } from 'react';
import axios from 'axios';

const AIInsights = ({ userId }) => {
  const [insights, setInsights] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchInsights();
  }, [userId]);

  const fetchInsights = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/users/${userId}/ai-insights`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setInsights(response.data.data);
    } catch (error) {
      console.error('Error fetching AI insights:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading insights...</div>;

  const getRiskColor = (level) => {
    const colors = { low: 'green', medium: 'orange', high: 'red' };
    return colors[level] || 'gray';
  };

  return (
    <div className="ai-insights">
      <h2>AI-Powered Insights</h2>
      
      <div className="insight-card">
        <h3>Risk Assessment</h3>
        <div className="risk-indicator" 
             style={{ backgroundColor: getRiskColor(insights.riskLevel) }}>
          {insights.riskLevel?.toUpperCase()}
        </div>
        <p>Risk Score: {insights.riskScore}</p>
        {insights.riskFactors?.length > 0 && (
          <ul>
            {insights.riskFactors.map((factor, idx) => (
              <li key={idx}>{factor}</li>
            ))}
          </ul>
        )}
      </div>

      <div className="insight-card">
        <h3>Burnout Analysis</h3>
        <div className="progress-bar">
          <div 
            className="progress-fill"
            style={{ 
              width: `${insights.burnoutScore * 100}%`,
              backgroundColor: insights.burnoutScore > 0.7 ? 'red' : 'green'
            }}
          />
        </div>
        <p>{insights.burnoutScore > 0.7 
          ? '⚠️ High burnout risk - Consider reducing workload'
          : '✅ Healthy workload balance'
        }</p>
      </div>

      <div className="insight-card">
        <h3>Performance Metrics</h3>
        <ul>
          <li>Tasks Completed: {insights.tasksCompleted}</li>
          <li>Average Work Hours: {insights.avgWorkHours}</li>
          <li>Completion Rate: {insights.completionRate}%</li>
        </ul>
      </div>
    </div>
  );
};

export default AIInsights;
```

## Common Patterns

### Admin User Creation
```javascript
const createUser = async (userData) => {
  const token = localStorage.getItem('token');
  const response = await axios.post(
    `${process.env.REACT_APP_API_URL}/api/users`,
    userData,
    {
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      }
    }
  );
  return response.data;
};
```

### Task Time Tracking
```javascript
const TaskTimer = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const saveTimeLog = async () => {
    const token = localStorage.getItem('token');
    await axios.post(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time-log`,
      { duration: seconds },
      { headers: { Authorization: `Bearer ${token}` } }
    );
  };

  return (
    <div>
      <p>{Math.floor(seconds / 3600)}h {Math.floor((seconds % 3600) / 60)}m {seconds % 60}s</p>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={saveTimeLog}>Save</button>
    </div>
  );
};
```

### Ticket Auto-Classification
```javascript
const submitTicket = async (ticketData) => {
  // First, get AI classification
  const classification = await axios.post(
    `${process.env.REACT_APP_ML_URL}/ml/classify-ticket`,
    {
      subject: ticketData.subject,
      description: ticketData.description
    }
  );

  // Then create ticket with AI-suggested category and priority
  const token = localStorage.getItem('token');
  const response = await axios.post(
    `${process.env.REACT_APP_API_URL}/api/tickets`,
    {
      ...ticketData,
      category: classification.data.category,
      priority: classification.data.priority,
      aiClassified: true
    },
    { headers: { Authorization: `Bearer ${token}` } }
  );

  return response.data;
};
```

## Configuration

### Backend Environment Variables
```env
PORT=5000
NODE_ENV=development
MONGODB_URI=mongodb://localhost:27017/enterprise_db
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
MAX_LOGIN_ATTEMPTS=5
SESSION_TIMEOUT=3600000
```

### Frontend Environment Variables
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_ENABLE_AI=true
REACT_APP_REFRESH_INTERVAL=30000
```

### ML Service Configuration
```env
MODEL_PATH=./models
LOG_LEVEL=info
ENABLE_TRAINING=true
BATCH_SIZE=32
PREDICTION_THRESHOLD=0.7
```

## Troubleshooting

### MongoDB Connection Issues
```bash
# Check MongoDB status
sudo systemctl status mongod

# Restart MongoDB
sudo systemctl restart mongod

# Check connection string in backend/.env
MONGODB_URI=mongodb://localhost:27017/enterprise_db
```

### JWT Authentication Errors
```javascript
// Verify token is being sent correctly
const token = localStorage.getItem('token');
if (!token) {
  console.error('No token found - user must log in');
  window.location.href = '/login';
}

// Check token expiration
const decoded = jwt.decode(token);
if (decoded.exp * 1000 < Date.now()) {
  console.error('Token expired - refreshing');
  // Implement token refresh logic
}
```

### ML Service Not Responding
```bash
# Check if service is running
curl http://localhost:8000/health

# Check logs
cd ml-service
tail -f logs/ml-service.log

# Restart service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### CORS Issues
```javascript
// Backend: Update CORS configuration
const cors = require('cors');
app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### Task Status Not Updating
```javascript
// Ensure you're sending correct status values
const validStatuses = ['todo', 'in-progress', 'done'];
if (!validStatuses.includes(newStatus)) {
  throw new Error('Invalid status value');
}

// Check backend validation
const Task = new Schema({
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  }
});
```

### AI Predictions Inaccurate
```python
# Retrain models with more data
from models.risk_detector import RiskDetector

detector = RiskDetector()
detector.train(training_data)
detector.save_model('./models/risk_detector_v2.pkl')

# Adjust prediction thresholds
BURNOUT_THRESHOLD = 0.7  # Adjust based on false positive rate
RISK_THRESHOLD = 0.6
```

## Production Deployment

### Environment Setup
```bash
# Backend
cd backend
npm install --production
NODE_ENV=production npm start

# Use PM2 for process management
pm2 start npm --name "enterprise-backend" -- start
pm2 save
pm2 startup

# ML Service with gunicorn
cd ml-service
gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app --bind 0.0.0.0:8000

# Frontend build
cd frontend
npm run build
# Serve build folder with nginx or similar
```

### Security Considerations
- Always use HTTPS in production
- Set strong JWT_SECRET (minimum 32 characters)
- Implement rate limiting on authentication endpoints
- Enable MongoDB authentication
- Regularly update dependencies
- Use environment variables for all secrets
- Implement API request throttling
- Enable audit logging for admin actions
