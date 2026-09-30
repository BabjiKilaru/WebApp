import { CommonModule } from '@angular/common';
import { HttpClient } from '@angular/common/http';
import {
  ChangeDetectorRef,
  Component,
  OnInit
} from '@angular/core';
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
  imports: [
    CommonModule,
    FormsModule
  ],
  templateUrl: './user-management.html',
  styleUrl: './user-management.css'
})
export class UserManagement implements OnInit {

  users: AdminUser[] = [];

  filteredUsers: AdminUser[] = [];

  searchText = '';

  loading = false;

  loadError = '';

  constructor(
    private http: HttpClient,
    private cdr: ChangeDetectorRef
  ) {}

  ngOnInit(): void {
    this.loadUsers();
  }

  loadUsers(): void {

    this.loading = true;
    this.loadError = '';

    this.http
      .get<AdminUser[]>(
        '/api/admin/users'
      )
      .subscribe({

        next: users => {

          this.users = users ?? [];

          this.applyFilter();

          this.loading = false;

          this.cdr.markForCheck();
        },

        error: error => {

          console.error(
            'Unable to load users:',
            error
          );

          this.loading = false;

          this.loadError =
            'Unable to load users.';

          this.cdr.markForCheck();
        }
      });
  }

  applyFilter(): void {

    const value =
      this.searchText
        .trim()
        .toLowerCase();

    if (!value) {
      this.filteredUsers = [
        ...this.users
      ];

      return;
    }

    this.filteredUsers =
      this.users.filter(user => {

        const searchValues = [
          user.userid,
          user.fname,
          user.lname,
          user.facility,
          user.access,
          user.clinType,
          user.purpose
        ];

        return searchValues.some(
          item =>
            (item ?? '')
              .toLowerCase()
              .includes(value)
        );
      });
  }

  editUser(
    user: AdminUser
  ): void {

    console.log(
      'Edit user:',
      user
    );
  }
}






<div class="user-management">

  <div class="page-header">

    <div>
      <h2>
        User Management
      </h2>

      <p>
        View and manage Gait Lab users.
      </p>
    </div>

    <div class="search-box">

      <input
        type="text"
        placeholder="Search users"
        [(ngModel)]="searchText"
        (ngModelChange)="applyFilter()"
      />

    </div>

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

          @for (
            user of filteredUsers;
            track user.userid
          ) {

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
                  {{
                    user.active
                      ? 'Active'
                      : 'Inactive'
                  }}
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

    <div class="user-count">
      Total users:
      {{ filteredUsers.length }}
    </div>

  }

</div>








.user-management {
  width: 100%;
}

.page-header {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 20px;
  margin-bottom: 20px;
}

.page-header h2 {
  margin: 0 0 5px;
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
  width: 280px;
}

.search-box input {
  width: 100%;
  height: 38px;
  padding: 0 12px;
  border: 1px solid #cfd8dc;
  border-radius: 6px;
  background: #ffffff;
  color: #263238;
  font-size: 14px;
  box-sizing: border-box;
  outline: none;
}

.search-box input:focus {
  border-color: #009688;
}

.table-container {
  width: 100%;
  overflow-x: auto;
  border: 1px solid #dfe5e8;
  border-radius: 8px;
  background: #ffffff;
}

table {
  width: 100%;
  min-width: 1050px;
  border-collapse: collapse;
}

thead {
  background: #f4f7f8;
}

th {
  padding: 12px 13px;
  border-bottom: 1px solid #dfe5e8;
  color: #455a64;
  font-size: 12px;
  font-weight: 700;
  text-align: left;
  white-space: nowrap;
}

td {
  padding: 11px 13px;
  border-bottom: 1px solid #edf0f2;
  color: #37474f;
  font-size: 13px;
  vertical-align: middle;
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
  width: 80px;
  text-align: center;
}

.edit-button {
  min-width: 58px;
  height: 31px;
  padding: 0 13px;
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
  min-width: 58px;
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
  padding: 24px;
  border: 1px solid #dfe5e8;
  border-radius: 8px;
  background: #ffffff;
  color: #607078;
  font-size: 14px;
  text-align: center;
}

.error-message {
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

.user-count {
  margin-top: 10px;
  color: #728087;
  font-size: 12px;
  text-align: right;
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
