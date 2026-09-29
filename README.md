.session-page {
  min-height: 100vh;
  background: #f5f7f8;
}

.session-content {
  width: 100%;
  max-width: 1000px;
  margin: 0 auto;
  padding: 28px 30px 70px;
  box-sizing: border-box;
}

.status-card {
  max-width: 700px;
  margin: 50px auto;
  padding: 30px;
  background: #ffffff;
  border: 1px solid #dfe5e8;
  border-radius: 10px;
  box-sizing: border-box;
}

.status-card h2 {
  margin: 0 0 12px;
  color: #263238;
  font-size: 22px;
  font-weight: 700;
}

.status-card p {
  margin: 0;
  color: #607078;
  font-size: 14px;
  line-height: 1.5;
}

.questionnaire-header {
  margin-bottom: 16px;
}

.progress {
  margin-bottom: 4px;
  color: #728087;
  font-size: 12px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.2px;
}

.questionnaire-header h1 {
  margin: 0;
  color: #263238;
  font-size: 25px;
  font-weight: 700;
}

.questionnaire-body {
  width: 100%;
  background: #ffffff;
  border: 1px solid #dfe5e8;
  border-radius: 10px;
  padding: 26px;
  box-sizing: border-box;
}

.questionnaire-actions {
  position: sticky;
  bottom: 0;
  z-index: 30;

  display: flex;
  align-items: center;
  justify-content: space-between;

  gap: 16px;

  width: 100%;
  padding: 16px 0;
  margin-top: 18px;

  background: #f5f7f8;

  box-sizing: border-box;
}

.action-message {
  display: flex;
  align-items: center;

  flex: 1;
  min-width: 0;
}

.bottom-error-message {
  display: inline-block;

  max-width: 100%;

  padding: 9px 13px;

  border: 1px solid #e3a9a9;
  border-radius: 6px;

  background: #fff3f3;
  color: #a12626;

  font-size: 13px;
  line-height: 1.4;

  box-sizing: border-box;
}

.action-buttons {
  display: flex;
  align-items: center;
  justify-content: flex-end;

  gap: 10px;

  flex-shrink: 0;
}

.primary-button,
.secondary-button {
  min-height: 40px;
  padding: 0 20px;

  border-radius: 6px;

  font-size: 14px;
  font-weight: 600;

  white-space: nowrap;

  cursor: pointer;

  transition:
    background 0.15s ease,
    border-color 0.15s ease,
    opacity 0.15s ease;
}

.primary-button {
  border: none;

  background: #009688;
  color: #ffffff;
}

.primary-button:hover:not(:disabled) {
  background: #00796b;
}

.secondary-button {
  border: 1px solid #aebbc0;

  background: #ffffff;
  color: #455a64;
}

.secondary-button:hover:not(:disabled) {
  background: #f3f6f7;
  border-color: #91a1a8;
}

.primary-button:disabled,
.secondary-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.error-message {
  margin-bottom: 16px;
  padding: 12px 15px;

  border: 1px solid #e3a9a9;
  border-radius: 6px;

  background: #fff3f3;
  color: #a12626;

  font-size: 13px;
  line-height: 1.4;
}

@media (max-width: 700px) {
  .session-content {
    padding: 18px 14px 50px;
  }

  .questionnaire-body {
    padding: 18px;
  }

  .questionnaire-actions {
    align-items: stretch;
    flex-direction: column;

    gap: 10px;

    padding: 12px 0;
  }

  .action-message {
    width: 100%;
  }

  .bottom-error-message {
    width: 100%;
  }

  .action-buttons {
    width: 100%;

    display: grid;
    grid-template-columns: 1fr 1fr;

    gap: 10px;
  }

  .primary-button,
  .secondary-button {
    width: 100%;
  }
}
