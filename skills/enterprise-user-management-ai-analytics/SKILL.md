---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "create user dashboard with task tracking"
  - "implement ticket classification with AI"
  - "add burnout detection to user system"
  - "build kanban board with time tracking"
  - "configure ML service for risk prediction"
  - "deploy user management with FastAPI backend"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack web application for managing enterprise users, tasks, and support tickets with integrated AI analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights.

## What It Does

This system provides:
- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-powered ticket classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Separate admin and user dashboards with real-time insights

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance

### 1. Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### 2. Backend Setup

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

npm start
```

### 3. ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=${MONGODB_URI}
MODEL_PATH=./models
EOF

uvicorn main:app --reload --port 8000
```

### 4. Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

npm start
```

## Project Structure

```
├── frontend/          # React.js application
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── services/
│   │   └── utils/
├── backend/           # Node.js REST API
│   ├── controllers/
│   ├── models/
│   ├── routes/
│   └── middleware/
└── ml-service/        # FastAPI ML service
    ├── models/
    ├── utils/
    └── main.py
```

## Backend API Usage

### Authentication

```javascript
// Register new user
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role // 'admin' or 'user'
    })
  });
  return await response.json();
};

// Login
const login = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

```javascript
// Get all users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};

// Update user
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};

// Delete user
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      status: 'todo', // 'todo', 'in-progress', 'done'
      priority: taskData.priority,
      dueDate: taskData.dueDate
    })
  });
  return await response.json();
};

// Update task status
const updateTaskStatus = async (taskId, status) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status })
  });
  return await response.json();
};

// Get user tasks
const getUserTasks = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### Ticket Management

```javascript
// Create support ticket
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return await response.json();
};

// Get all tickets (Admin)
const getAllTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

## ML Service API Usage

### Risk Prediction

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np

app = FastAPI()

class RiskPredictionRequest(BaseModel):
    user_id: str
    failed_logins: int
    unusual_hours_activity: int
    data_access_volume: float
    permission_changes: int

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Load trained model
        model = joblib.load('./models/risk_model.pkl')
        
        features = np.array([[
            request.failed_logins,
            request.unusual_hours_activity,
            request.data_access_volume,
            request.permission_changes
        ]])
        
        risk_score = model.predict_proba(features)[0][1]
        risk_level = "High" if risk_score > 0.7 else "Medium" if risk_score > 0.4 else "Low"
        
        return {
            "user_id": request.user_id,
            "risk_score": float(risk_score),
            "risk_level": risk_level,
            "recommendations": get_recommendations(risk_level)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

```javascript
// Frontend: Call ML service
const predictUserRisk = async (userId, metrics) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      failed_logins: metrics.failedLogins,
      unusual_hours_activity: metrics.unusualHours,
      data_access_volume: metrics.dataAccess,
      permission_changes: metrics.permissionChanges
    })
  });
  return await response.json();
};
```

### Burnout Detection

```python
# ml-service/main.py
class BurnoutRequest(BaseModel):
    user_id: str
    hours_worked: float
    tasks_completed: int
    tasks_overdue: int
    weekend_work_hours: float
    avg_task_completion_time: float

@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutRequest):
    try:
        model = joblib.load('./models/burnout_model.pkl')
        
        features = np.array([[
            request.hours_worked,
            request.tasks_completed,
            request.tasks_overdue,
            request.weekend_work_hours,
            request.avg_task_completion_time
        ]])
        
        burnout_probability = model.predict_proba(features)[0][1]
        burnout_risk = "High" if burnout_probability > 0.6 else "Medium" if burnout_probability > 0.3 else "Low"
        
        return {
            "user_id": request.user_id,
            "burnout_probability": float(burnout_probability),
            "burnout_risk": burnout_risk,
            "suggestions": get_burnout_suggestions(burnout_risk)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Ticket Classification

```python
# ml-service/main.py
from sklearn.feature_extraction.text import TfidfVectorizer

class TicketClassificationRequest(BaseModel):
    ticket_id: str
    title: str
    description: str

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        # Load models
        vectorizer = joblib.load('./models/ticket_vectorizer.pkl')
        classifier = joblib.load('./models/ticket_classifier.pkl')
        
        # Combine title and description
        text = f"{request.title} {request.description}"
        
        # Transform and predict
        features = vectorizer.transform([text])
        category = classifier.predict(features)[0]
        priority = predict_priority(features)
        
        return {
            "ticket_id": request.ticket_id,
            "category": category,
            "priority": priority,
            "estimated_resolution_time": estimate_resolution_time(category, priority)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Frontend Components

### Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    });
  };

  const onDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const onDrop = async (e, newStatus) => {
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');
    
    if (currentStatus !== newStatus) {
      await updateTaskStatus(taskId, newStatus);
      fetchTasks();
    }
  };

  const updateTaskStatus = async (taskId, status) => {
    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status })
    });
  };

  return (
    <div className="kanban-board">
      <Column
        title="To Do"
        tasks={tasks.todo}
        status="todo"
        onDragStart={onDragStart}
        onDrop={onDrop}
      />
      <Column
        title="In Progress"
        tasks={tasks.inProgress}
        status="in-progress"
        onDragStart={onDragStart}
        onDrop={onDrop}
      />
      <Column
        title="Done"
        tasks={tasks.done}
        status="done"
        onDragStart={onDragStart}
        onDrop={onDrop}
      />
    </div>
  );
};

const Column = ({ title, tasks, status, onDragStart, onDrop }) => (
  <div
    className="kanban-column"
    onDragOver={(e) => e.preventDefault()}
    onDrop={(e) => onDrop(e, status)}
  >
    <h3>{title}</h3>
    {tasks.map(task => (
      <div
        key={task._id}
        className="task-card"
        draggable
        onDragStart={(e) => onDragStart(e, task._id, status)}
      >
        <h4>{task.title}</h4>
        <p>{task.description}</p>
        <span className={`priority-${task.priority}`}>{task.priority}</span>
      </div>
    ))}
  </div>
);

export default KanbanBoard;
```

### Time Tracker Component

```javascript
// frontend/src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isActive, setIsActive] = useState(false);

  useEffect(() => {
    let interval = null;
    if (isActive) {
      interval = setInterval(() => {
        setSeconds(seconds => seconds + 1);
      }, 1000);
    } else if (!isActive && seconds !== 0) {
      clearInterval(interval);
    }
    return () => clearInterval(interval);
  }, [isActive, seconds]);

  const toggle = () => {
    setIsActive(!isActive);
  };

  const reset = async () => {
    setSeconds(0);
    setIsActive(false);
    
    // Save time to backend
    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ timeSpent: seconds })
    });
  };

  const formatTime = (seconds) => {
    const hours = Math.floor(seconds / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={toggle}>{isActive ? 'Pause' : 'Start'}</button>
      <button onClick={reset}>Save & Reset</button>
    </div>
  );
};

export default TimeTracker;
```

### AI Insights Dashboard

```javascript
// frontend/src/components/AIInsights.jsx
import React, { useState, useEffect } from 'react';

const AIInsights = ({ userId }) => {
  const [insights, setInsights] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAIInsights();
  }, [userId]);

  const fetchAIInsights = async () => {
    try {
      const token = localStorage.getItem('token');
      
      // Fetch user metrics
      const metricsResponse = await fetch(`http://localhost:5000/api/users/${userId}/metrics`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const metrics = await metricsResponse.json();
      
      // Get risk prediction
      const riskResponse = await fetch('http://localhost:8000/api/ml/predict-risk', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          user_id: userId,
          failed_logins: metrics.failedLogins,
          unusual_hours_activity: metrics.unusualHours,
          data_access_volume: metrics.dataAccess,
          permission_changes: metrics.permissionChanges
        })
      });
      const riskData = await riskResponse.json();
      
      // Get burnout detection
      const burnoutResponse = await fetch('http://localhost:8000/api/ml/detect-burnout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          user_id: userId,
          hours_worked: metrics.hoursWorked,
          tasks_completed: metrics.tasksCompleted,
          tasks_overdue: metrics.tasksOverdue,
          weekend_work_hours: metrics.weekendHours,
          avg_task_completion_time: metrics.avgCompletionTime
        })
      });
      const burnoutData = await burnoutResponse.json();
      
      setInsights({ risk: riskData, burnout: burnoutData });
      setLoading(false);
    } catch (error) {
      console.error('Error fetching AI insights:', error);
      setLoading(false);
    }
  };

  if (loading) return <div>Loading insights...</div>;

  return (
    <div className="ai-insights">
      <div className="insight-card">
        <h3>Security Risk Analysis</h3>
        <div className={`risk-level ${insights.risk.risk_level.toLowerCase()}`}>
          {insights.risk.risk_level} Risk
        </div>
        <p>Risk Score: {(insights.risk.risk_score * 100).toFixed(1)}%</p>
        <ul>
          {insights.risk.recommendations.map((rec, idx) => (
            <li key={idx}>{rec}</li>
          ))}
        </ul>
      </div>
      
      <div className="insight-card">
        <h3>Burnout Detection</h3>
        <div className={`burnout-risk ${insights.burnout.burnout_risk.toLowerCase()}`}>
          {insights.burnout.burnout_risk} Risk
        </div>
        <p>Burnout Probability: {(insights.burnout.burnout_probability * 100).toFixed(1)}%</p>
        <ul>
          {insights.burnout.suggestions.map((sug, idx) => (
            <li key={idx}>{sug}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

export default AIInsights;
```

## Common Patterns

### Protected Routes

```javascript
// frontend/src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { jwtDecode } from 'jwt-decode';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  try {
    const decoded = jwtDecode(token);
    
    if (requiredRole && decoded.role !== requiredRole) {
      return <Navigate to="/unauthorized" />;
    }
    
    return children;
  } catch (error) {
    return <Navigate to="/login" />;
  }
};

export default ProtectedRoute;
```

### Real-time Notifications

```javascript
// frontend/src/services/notificationService.js
class NotificationService {
  constructor() {
    this.ws = null;
  }

  connect(userId) {
    this.ws = new WebSocket(`ws://localhost:5000/notifications/${userId}`);
    
    this.ws.onmessage = (event) => {
      const notification = JSON.parse(event.data);
      this.handleNotification(notification);
    };
  }

  handleNotification(notification) {
    // Show browser notification
    if (Notification.permission === 'granted') {
      new Notification(notification.title, {
        body: notification.message,
        icon: '/notification-icon.png'
      });
    }
    
    // Update UI
    const event = new CustomEvent('notification', { detail: notification });
    window.dispatchEvent(event);
  }

  disconnect() {
    if (this.ws) {
      this.ws.close();
    }
  }
}

export default new NotificationService();
```

## Training ML Models

### Risk Prediction Model

```python
# ml-service/train_models.py
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
import joblib

def train_risk_model():
    # Load training data
    data = pd.read_csv('./data/user_risk_data.csv')
    
    X = data[['failed_logins', 'unusual_hours_activity', 
              'data_access_volume', 'permission_changes']]
    y = data['is_risky']
    
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )
    
    model = RandomForestClassifier(n_estimators=100, random_state=42)
    model.fit(X_train, y_train)
    
    accuracy = model.score(X_test, y_test)
    print(f"Risk model accuracy: {accuracy:.2f}")
    
    joblib.dump(model, './models/risk_model.pkl')

if __name__ == "__main__":
    train_risk_model()
```

## Configuration

### Environment Variables

**Backend (.env)**
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_mgmt
JWT_SECRET=your-secret-key-here
JWT_EXPIRES_IN=24h
ML_SERVICE_URL=http://localhost:8000
ENABLE_AUDIT_LOGS=true
```

**ML Service (.env)**
```env
MONGODB_URI=mongodb://localhost:27017/enterprise_mgmt
MODEL_PATH=./models
ENABLE_ONLINE_LEARNING=true
LOG_LEVEL=INFO
```

**Frontend (.env)**
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_WS_URL=ws://localhost:5000
```

## Troubleshooting

### JWT Token Expired
```javascript
// Implement token refresh
const refreshToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  const response = await fetch('http://localhost:5000/api/auth/refresh', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken })
  });
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data.token;
};
```

### ML Model Not Loading
```python
# Check model file exists
import os
from pathlib import Path

def check_models():
    model_path = Path('./models')
    required_models = ['risk_model.pkl', 'burnout_model.pkl', 'ticket_classifier.pkl']
    
    for model in required_models:
        if not (model_path / model).exists():
            print(f"Missing model: {model}")
            print("Run: python train_models.py")
            return False
    return True
```

### MongoDB Connection Issues
```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Issues
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

## Performance Optimization

### Caching User Data
```javascript
// Backend: Implement Redis caching
const redis = require('redis');
const client = redis.createClient();

const getCachedUser = async (userId) => {
  const cached = await client.get(`user:${userId}`);
  if (cached) return JSON.parse(cached);
  
  const user = await User.findById(userId);
  await client.setex(`user:${userId}`, 3600, JSON.stringify(user));
  return user;
};
```

### Batch ML Predictions
```python
# Process multiple predictions in batch
@app.post("/api/ml/batch-predict-risk")
async def batch_predict_risk(requests: List[RiskPredictionRequest]):
    model = joblib.load('./models/risk_model.pkl')
    
    features = np.array([[
        req.failed_logins,
        req.unusual_hours_activity,
        req.data_access_volume,
        req.permission_changes
    ] for req in requests])
    
    predictions = model.predict_proba(features)
    
    return [
        {
            "user_id": req.user_id,
            "risk_score": float(pred[1]),
            "risk_level": "High" if pred[1] > 0.7 else "Medium" if pred[1] > 0.4 else "Low"
        }
        for req, pred in zip(requests, predictions)
    ]
```
