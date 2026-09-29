:host {
  display: block;

  min-height: 100vh;

  background: #f7f9fc;

  color: #173763;
}

.admin-page {
  min-height: 100vh;
}

.admin-layout {
  display: grid;

  grid-template-columns:
    300px minmax(0, 1fr);

  gap: 18px;

  padding:
    18px 24px 28px;
}

.admin-sidebar {
  background: #ffffff;

  border:
    1px solid #e1e7ef;

  border-radius: 8px;

  overflow: hidden;
}

.admin-title {
  padding:
    20px 22px;

  color: #173763;

  font-size: 22px;
  font-weight: 700;

  border-bottom:
    1px solid #e4e9f0;
}

.admin-menu {
  display: flex;
  flex-direction: column;
}

.admin-menu-item {
  display: flex;
  align-items: center;
  justify-content: space-between;

  width: 100%;

  min-height: 52px;

  padding:
    0 20px;

  color: #415774;

  background: #ffffff;

  border: none;
  border-bottom:
    1px solid #edf1f5;

  font: inherit;

  font-size: 15px;
  font-weight: 600;

  text-align: left;

  cursor: pointer;
}

.admin-menu-item:hover {
  background: #f6faf9;
}

.admin-menu-item.active {
  color: #007f75;

  background: #eef8f7;

  border-left:
    4px solid #009688;

  padding-left: 16px;
}

.menu-arrow {
  color: #506682;

  font-size: 24px;
  line-height: 1;
}

.lookup-menu {
  max-height: 480px;

  overflow-y: auto;

  padding:
    6px 0 10px;

  background: #fbfcfd;
}

.lookup-menu button {
  display: block;

  width: 100%;

  padding:
    9px 20px 9px 32px;

  color: #53677f;

  background: transparent;

  border: none;

  font: inherit;

  font-size: 13px;

  text-align: left;

  cursor: pointer;
}

.lookup-menu button:hover {
  color: #007f75;

  background: #eef8f7;
}

.admin-content {
  min-height: 650px;

  padding:
    22px 24px;

  background: #ffffff;

  border:
    1px solid #e1e7ef;

  border-radius: 8px;
}

.content-header {
  display: flex;
  align-items: center;
  justify-content: space-between;

  padding-bottom: 16px;

  border-bottom:
    1px solid #e4e9f0;
}

.content-header h2 {
  margin: 0;

  color: #173763;

  font-size: 21px;
  font-weight: 700;
}

.primary-button {
  height: 38px;

  padding:
    0 16px;

  color: #ffffff;

  background: #009688;

  border:
    1px solid #009688;

  border-radius: 6px;

  font: inherit;

  font-size: 13px;
  font-weight: 600;

  cursor: pointer;
}

.primary-button:hover {
  background: #00796b;
}

.empty-content {
  display: flex;
  align-items: center;
  justify-content: center;

  min-height: 480px;

  color: #8593a6;

  font-size: 14px;
}

@media (
  max-width: 900px
) {

  .admin-layout {
    grid-template-columns:
      240px minmax(0, 1fr);
  }
}
