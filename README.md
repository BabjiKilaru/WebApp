import { CommonModule } from '@angular/common';
import { Component } from '@angular/core';
import { Header } from '../header/header';
import { UserManagement } from './user-management/user-management';

type AdminSection = 'users' | 'lookups' | 'reports';

@Component({
    selector: 'app-admin',
    standalone: true,
    imports: [CommonModule, Header, UserManagement],
    templateUrl: './admin.html',
    styleUrl: './admin.css'
})
export class Admin {
    activeSection: AdminSection = 'users';
    lookupExpanded = false;

    selectSection(section: AdminSection): void {
        this.activeSection = section;

        if (section !== 'lookups') {
            this.lookupExpanded = false;
        }
    }

    toggleLookups(): void {
        this.activeSection = 'lookups';
        this.lookupExpanded = !this.lookupExpanded;
    }
}






<div class="admin-page">
    <app-header></app-header>

    <main class="admin-layout">
        <aside class="admin-sidebar">
            <div class="admin-title">
                Admin Menu
            </div>

            <nav class="admin-menu">
                <button
                    type="button"
                    class="admin-menu-item"
                    [class.active]="activeSection === 'users'"
                    (click)="selectSection('users')"
                >
                    <span>
                        User Management
                    </span>

                    <span class="menu-arrow">
                        ›
                    </span>
                </button>

                <button
                    type="button"
                    class="admin-menu-item"
                    [class.active]="activeSection === 'lookups'"
                    (click)="toggleLookups()"
                >
                    <span>
                        Edit Lookup Tables
                    </span>

                    <span
                        class="menu-arrow lookup-arrow"
                        [class.expanded]="lookupExpanded"
                    >
                        ›
                    </span>
                </button>

                @if (lookupExpanded) {
                    <div class="lookup-menu">
                        <button type="button">
                            Body Locations for Botox Injections
                        </button>

                        <button type="button">
                            Orthoses
                        </button>

                        <button type="button">
                            Clinician Types
                        </button>

                        <button type="button">
                            Devices
                        </button>

                        <button type="button">
                            Gait Concerns
                        </button>

                        <button type="button">
                            Health Conditions
                        </button>

                        <button type="button">
                            History Conditions/Events
                        </button>

                        <button type="button">
                            ICD-9 (Diagnoses)
                        </button>

                        <button type="button">
                            Loops
                        </button>

                        <button type="button">
                            Keywords
                        </button>

                        <button type="button">
                            Motor Control Tests
                        </button>

                        <button type="button">
                            Muscles for EMG Analysis
                        </button>

                        <button type="button">
                            NonPT Interp Choices
                        </button>

                        <button type="button">
                            NonPT Interp Fields
                        </button>

                        <button type="button">
                            Pain Locations
                        </button>

                        <button type="button">
                            PROM Tests
                        </button>

                        <button type="button">
                            Relationship List
                        </button>

                        <button type="button">
                            Seizure Medication List
                        </button>

                        <button type="button">
                            Strength Tests
                        </button>

                        <button type="button">
                            Surgery List
                        </button>

                        <button type="button">
                            Tests
                        </button>

                        <button type="button">
                            Tone Tests
                        </button>

                        <button type="button">
                            Visit Types
                        </button>

                        <button type="button">
                            Visit Subtypes
                        </button>
                    </div>
                }

                <button
                    type="button"
                    class="admin-menu-item"
                    [class.active]="activeSection === 'reports'"
                    (click)="selectSection('reports')"
                >
                    <span>
                        Reports
                    </span>

                    <span class="menu-arrow">
                        ›
                    </span>
                </button>
            </nav>
        </aside>

        <section class="admin-content">
            @if (activeSection === 'users') {
                <app-user-management></app-user-management>
            }

            @if (activeSection === 'lookups') {
                <div class="simple-content-page">
                    <div class="content-header">
                        <h2>
                            Edit Lookup Tables
                        </h2>
                    </div>

                    <div class="empty-content">
                        Select a lookup table from the menu.
                    </div>
                </div>
            }

            @if (activeSection === 'reports') {
                <div class="simple-content-page">
                    <div class="content-header">
                        <h2>
                            Reports
                        </h2>
                    </div>

                    <div class="empty-content">
                        Reports content page
                    </div>
                </div>
            }
        </section>
    </main>
</div>








:host {
  display: block;
  height: 100vh;
  overflow: hidden;
  background: #f7f9fc;
  color: #173763;
}

.admin-page {
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
}

app-header {
  flex: 0 0 auto;
}

.admin-layout {
  display: grid;
  grid-template-columns: 250px minmax(0, 1fr);
  gap: 16px;
  flex: 1 1 auto;
  min-height: 0;
  padding: 16px 20px 20px;
  overflow: hidden;
  box-sizing: border-box;
}

.admin-sidebar {
  display: flex;
  flex-direction: column;
  min-height: 0;
  height: 100%;
  background: #ffffff;
  border: 1px solid #e1e7ef;
  border-radius: 8px;
  overflow: hidden;
}

.admin-title {
  flex: 0 0 auto;
  padding: 18px 18px;
  color: #173763;
  font-size: 20px;
  font-weight: 700;
  border-bottom: 1px solid #e4e9f0;
}

.admin-menu {
  display: flex;
  flex-direction: column;
  flex: 1 1 auto;
  min-height: 0;
  overflow-y: auto;
  overflow-x: hidden;
}

.admin-menu-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  min-height: 48px;
  padding: 0 16px;
  color: #415774;
  background: #ffffff;
  border: none;
  border-bottom: 1px solid #edf1f5;
  font: inherit;
  font-size: 14px;
  font-weight: 600;
  text-align: left;
  cursor: pointer;
  box-sizing: border-box;
}

.admin-menu-item:hover {
  background: #f6faf9;
}

.admin-menu-item.active {
  color: #007f75;
  background: #eef8f7;
  border-left: 4px solid #009688;
  padding-left: 12px;
}

.menu-arrow {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 18px;
  height: 18px;
  margin-left: 10px;
  color: #66788d;
  font-size: 18px;
  font-weight: 400;
  line-height: 1;
  transform: translateY(-1px);
}

.lookup-arrow {
  transition: transform 0.18s ease;
}

.lookup-arrow.expanded {
  transform: rotate(90deg);
}

.lookup-menu {
  flex: 0 0 auto;
  background: #fbfcfd;
  border-bottom: 1px solid #edf1f5;
}

.lookup-menu button {
  display: block;
  width: 100%;
  min-height: 36px;
  padding: 8px 16px 8px 28px;
  color: #53677f;
  background: transparent;
  border: none;
  font: inherit;
  font-size: 12px;
  text-align: left;
  cursor: pointer;
}

.lookup-menu button:hover {
  color: #007f75;
  background: #eef8f7;
}

.admin-content {
  min-width: 0;
  min-height: 0;
  height: 100%;
  overflow: hidden;
  background: #ffffff;
  border: 1px solid #e1e7ef;
  border-radius: 8px;
}

.simple-content-page {
  display: flex;
  flex-direction: column;
  height: 100%;
  min-height: 0;
  padding: 20px 22px;
  box-sizing: border-box;
}

.content-header {
  flex: 0 0 auto;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding-bottom: 16px;
  border-bottom: 1px solid #e4e9f0;
}

.content-header h2 {
  margin: 0;
  color: #173763;
  font-size: 21px;
  font-weight: 700;
}

.empty-content {
  display: flex;
  align-items: center;
  justify-content: center;
  flex: 1 1 auto;
  min-height: 0;
  color: #8593a6;
  font-size: 14px;
}

.admin-menu::-webkit-scrollbar {
  width: 7px;
}

.admin-menu::-webkit-scrollbar-track {
  background: transparent;
}

.admin-menu::-webkit-scrollbar-thumb {
  background: #cfd8df;
  border-radius: 10px;
}

.admin-menu::-webkit-scrollbar-thumb:hover {
  background: #b8c4cc;
}

@media (max-width: 900px) {
  .admin-layout {
    grid-template-columns: 220px minmax(0, 1fr);
    padding: 12px;
    gap: 12px;
  }
}








import { CommonModule } from '@angular/common';
import { HttpClient } from '@angular/common/http';
import { ChangeDetectorRef, Component, OnInit } from '@angular/core';
import { FormsModule } from '@angular/forms';

interface AdminUser {
    userid: string;
    fname: string | null;
    lname: string | null;
    facility: string | null;
    access: string | null;
    userAdmin: boolean;
    clinType: string | null;
    purpose: string | null;
    active: boolean;
}

@Component({
    selector: 'app-user-management',
    standalone: true,
    imports: [CommonModule, FormsModule],
    templateUrl: './user-management.html',
    styleUrl: './user-management.css'
})
export class UserManagement implements OnInit {
    users: AdminUser[] = [];
    filteredUsers: AdminUser[] = [];

    searchText = '';
    accessFilter = '';
    adminFilter = '';
    clinTypeFilter = '';
    activeFilter = '';

    accessOptions: string[] = [];
    clinTypeOptions: string[] = [];

    loading = false;
    loadError = '';

    constructor(
        private http: HttpClient,
        private cdr: ChangeDetectorRef
    ) {
    }

    ngOnInit(): void {
        this.loadUsers();
    }

    loadUsers(): void {
        this.loading = true;
        this.loadError = '';

        this.http
            .get<AdminUser[]>('/api/admin/users')
            .subscribe({
                next: users => {
                    this.users = users ?? [];

                    this.accessOptions = this.getUniqueValues(
                        this.users.map(user => user.access)
                    );

                    this.clinTypeOptions = this.getUniqueValues(
                        this.users.map(user => user.clinType)
                    );

                    this.applyFilter();
                    this.loading = false;
                    this.cdr.markForCheck();
                },

                error: error => {
                    console.error('Unable to load users:', error);
                    this.users = [];
                    this.filteredUsers = [];
                    this.loading = false;
                    this.loadError = 'Unable to load users.';
                    this.cdr.markForCheck();
                }
            });
    }

    applyFilter(): void {
        const searchValue = this.searchText.trim().toLowerCase();

        this.filteredUsers = this.users.filter(user => {
            const searchValues = [
                user.userid,
                user.fname,
                user.lname,
                user.facility,
                user.access,
                user.clinType,
                user.purpose
            ];

            const matchesSearch =
                !searchValue ||
                searchValues.some(value =>
                    (value ?? '').toLowerCase().includes(searchValue)
                );

            const matchesAccess =
                !this.accessFilter ||
                user.access === this.accessFilter;

            const matchesAdmin =
                !this.adminFilter ||
                (this.adminFilter === 'yes' && user.userAdmin) ||
                (this.adminFilter === 'no' && !user.userAdmin);

            const matchesClinType =
                !this.clinTypeFilter ||
                user.clinType === this.clinTypeFilter;

            const matchesActive =
                !this.activeFilter ||
                (this.activeFilter === 'active' && user.active) ||
                (this.activeFilter === 'inactive' && !user.active);

            return (
                matchesSearch &&
                matchesAccess &&
                matchesAdmin &&
                matchesClinType &&
                matchesActive
            );
        });
    }

    clearFilters(): void {
        this.searchText = '';
        this.accessFilter = '';
        this.adminFilter = '';
        this.clinTypeFilter = '';
        this.activeFilter = '';

        this.applyFilter();
    }

    getUniqueValues(values: Array<string | null>): string[] {
        const filteredValues = values
            .filter((value): value is string => {
                return value !== null && value.trim() !== '';
            })
            .map(value => value.trim());

        return [...new Set(filteredValues)].sort((a, b) =>
            a.localeCompare(b)
        );
    }

    editUser(user: AdminUser): void {
        console.log('Edit user:', user);
    }
}










<div class="user-management">
    <div class="page-header">
        <div class="heading-block">
            <h2>
                User Management
            </h2>

            <p>
                View and manage Gait Lab users.
            </p>
        </div>

        <div class="search-box">
            <label for="userSearch">
                Search Users
            </label>

            <input
                id="userSearch"
                type="text"
                placeholder="Search users"
                [(ngModel)]="searchText"
                (ngModelChange)="applyFilter()"
            />
        </div>
    </div>

    <div class="filters">
        <div class="filter-field">
            <label for="accessFilter">
                Access
            </label>

            <select
                id="accessFilter"
                [(ngModel)]="accessFilter"
                (ngModelChange)="applyFilter()"
            >
                <option value="">
                    All
                </option>

                @for (access of accessOptions; track access) {
                    <option [value]="access">
                        {{ access }}
                    </option>
                }
            </select>
        </div>

        <div class="filter-field">
            <label for="adminFilter">
                Admin
            </label>

            <select
                id="adminFilter"
                [(ngModel)]="adminFilter"
                (ngModelChange)="applyFilter()"
            >
                <option value="">
                    All
                </option>

                <option value="yes">
                    Yes
                </option>

                <option value="no">
                    No
                </option>
            </select>
        </div>

        <div class="filter-field">
            <label for="clinTypeFilter">
                Clin Type
            </label>

            <select
                id="clinTypeFilter"
                [(ngModel)]="clinTypeFilter"
                (ngModelChange)="applyFilter()"
            >
                <option value="">
                    All
                </option>

                @for (clinType of clinTypeOptions; track clinType) {
                    <option [value]="clinType">
                        {{ clinType }}
                    </option>
                }
            </select>
        </div>

        <div class="filter-field">
            <label for="activeFilter">
                Active
            </label>

            <select
                id="activeFilter"
                [(ngModel)]="activeFilter"
                (ngModelChange)="applyFilter()"
            >
                <option value="">
                    All
                </option>

                <option value="active">
                    Active
                </option>

                <option value="inactive">
                    Inactive
                </option>
            </select>
        </div>

        <div class="filter-actions">
            <button
                type="button"
                class="clear-button"
                (click)="clearFilters()"
            >
                Clear Filters
            </button>
        </div>
    </div>

    <div class="results-row">
        <span>
            Showing {{ filteredUsers.length }} of {{ users.length }} users
        </span>
    </div>

    @if (loading) {
        <div class="status-message">
            Loading users...
        </div>
    } @else if (loadError) {
        <div class="error-message">
            {{ loadError }}
        </div>
    } @else {
        <div class="table-container">
            <table>
                <thead>
                    <tr>
                        <th>
                            User ID
                        </th>

                        <th>
                            First Name
                        </th>

                        <th>
                            Last Name
                        </th>

                        <th>
                            Facility
                        </th>

                        <th>
                            Access
                        </th>

                        <th>
                            Admin
                        </th>

                        <th>
                            Clin Type
                        </th>

                        <th>
                            Purpose
                        </th>

                        <th>
                            Active
                        </th>

                        <th class="actions-column">
                            Action
                        </th>
                    </tr>
                </thead>

                <tbody>
                    @for (user of filteredUsers; track user.userid) {
                        <tr>
                            <td class="userid">
                                {{ user.userid }}
                            </td>

                            <td>
                                {{ user.fname || '-' }}
                            </td>

                            <td>
                                {{ user.lname || '-' }}
                            </td>

                            <td>
                                {{ user.facility || '-' }}
                            </td>

                            <td>
                                {{ user.access || '-' }}
                            </td>

                            <td>
                                {{ user.userAdmin ? 'Yes' : 'No' }}
                            </td>

                            <td>
                                {{ user.clinType || '-' }}
                            </td>

                            <td>
                                {{ user.purpose || '-' }}
                            </td>

                            <td>
                                <span
                                    class="status-badge"
                                    [class.inactive]="!user.active"
                                >
                                    {{ user.active ? 'Active' : 'Inactive' }}
                                </span>
                            </td>

                            <td class="actions-column">
                                <button
                                    type="button"
                                    class="edit-button"
                                    (click)="editUser(user)"
                                >
                                    Edit
                                </button>
                            </td>
                        </tr>
                    } @empty {
                        <tr>
                            <td
                                colspan="10"
                                class="empty-row"
                            >
                                No users found.
                            </td>
                        </tr>
                    }
                </tbody>
            </table>
        </div>
    }
</div>









:host {
  display: block;
  height: 100%;
  min-height: 0;
}

.user-management {
  display: flex;
  flex-direction: column;
  height: 100%;
  min-height: 0;
  padding: 18px 20px;
  box-sizing: border-box;
  overflow: hidden;
}

.page-header {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  flex: 0 0 auto;
  gap: 20px;
  margin-bottom: 14px;
}

.heading-block {
  min-width: 0;
}

.page-header h2 {
  margin: 0 0 4px;
  color: #263238;
  font-size: 22px;
  font-weight: 700;
}

.page-header p {
  margin: 0;
  color: #728087;
  font-size: 13px;
}

.search-box {
  display: flex;
  flex-direction: column;
  width: 250px;
}

.search-box label,
.filter-field label {
  margin-bottom: 5px;
  color: #56666d;
  font-size: 12px;
  font-weight: 600;
}

.search-box input {
  width: 100%;
  height: 36px;
  padding: 0 11px;
  border: 1px solid #cfd8dc;
  border-radius: 6px;
  background: #ffffff;
  color: #263238;
  font-size: 13px;
  box-sizing: border-box;
  outline: none;
}

.search-box input:focus {
  border-color: #009688;
}

.filters {
  display: flex;
  align-items: flex-end;
  flex: 0 0 auto;
  gap: 12px;
  padding: 12px 14px;
  margin-bottom: 8px;
  background: #f8fafb;
  border: 1px solid #e2e7ea;
  border-radius: 7px;
}

.filter-field {
  display: flex;
  flex-direction: column;
  min-width: 130px;
}

.filter-field select {
  width: 100%;
  height: 34px;
  padding: 0 30px 0 10px;
  border: 1px solid #cfd8dc;
  border-radius: 6px;
  background: #ffffff;
  color: #37474f;
  font-size: 13px;
  outline: none;
  cursor: pointer;
}

.filter-field select:focus {
  border-color: #009688;
}

.filter-actions {
  display: flex;
  align-items: flex-end;
  margin-left: auto;
}

.clear-button {
  height: 34px;
  padding: 0 14px;
  border: 1px solid #b8c4ca;
  border-radius: 6px;
  background: #ffffff;
  color: #54656c;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
}

.clear-button:hover {
  background: #f1f5f6;
}

.results-row {
  flex: 0 0 auto;
  display: flex;
  justify-content: flex-end;
  margin-bottom: 7px;
  color: #728087;
  font-size: 12px;
}

.table-container {
  flex: 1 1 auto;
  min-height: 0;
  width: 100%;
  overflow: auto;
  border: 1px solid #dfe5e8;
  border-radius: 7px;
  background: #ffffff;
}

table {
  width: 100%;
  min-width: 1120px;
  border-collapse: separate;
  border-spacing: 0;
}

thead {
  background: #f4f7f8;
}

th {
  position: sticky;
  top: 0;
  z-index: 2;
  padding: 11px 12px;
  background: #f4f7f8;
  border-right: 1px solid #d9e0e4;
  border-bottom: 1px solid #cfd8dc;
  color: #455a64;
  font-size: 12px;
  font-weight: 700;
  text-align: left;
  white-space: nowrap;
}

td {
  padding: 10px 12px;
  border-right: 1px solid #e2e7ea;
  border-bottom: 1px solid #edf0f2;
  color: #37474f;
  font-size: 12px;
  vertical-align: middle;
  white-space: nowrap;
}

th:last-child,
td:last-child {
  border-right: none;
}

tbody tr:hover {
  background: #f8fbfb;
}

tbody tr:last-child td {
  border-bottom: none;
}

.userid {
  font-weight: 600;
}

.actions-column {
  width: 78px;
  text-align: center;
}

.edit-button {
  min-width: 56px;
  height: 29px;
  padding: 0 12px;
  border: 1px solid #009688;
  border-radius: 5px;
  background: #ffffff;
  color: #00796b;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
}

.edit-button:hover {
  background: #e6f4f2;
}

.status-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 54px;
  padding: 4px 8px;
  border-radius: 12px;
  background: #e3f3f1;
  color: #00796b;
  font-size: 11px;
  font-weight: 600;
}

.status-badge.inactive {
  background: #f1f3f4;
  color: #7a858a;
}

.status-message {
  flex: 1 1 auto;
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 0;
  border: 1px solid #dfe5e8;
  border-radius: 8px;
  background: #ffffff;
  color: #607078;
  font-size: 14px;
}

.error-message {
  flex: 0 0 auto;
  padding: 12px 15px;
  border: 1px solid #e3a9a9;
  border-radius: 6px;
  background: #fff3f3;
  color: #a12626;
  font-size: 13px;
}

.empty-row {
  padding: 30px;
  color: #7a878c;
  text-align: center;
}

.table-container::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

.table-container::-webkit-scrollbar-track {
  background: #f4f6f7;
}

.table-container::-webkit-scrollbar-thumb {
  background: #c7d0d5;
  border-radius: 10px;
}

.table-container::-webkit-scrollbar-thumb:hover {
  background: #aebbc2;
}

@media (max-width: 1000px) {
  .filters {
    overflow-x: auto;
  }

  .filter-field {
    min-width: 120px;
  }
}

@media (max-width: 800px) {
  .page-header {
    align-items: stretch;
    flex-direction: column;
  }

  .search-box {
    width: 100%;
  }
}









