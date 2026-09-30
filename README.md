package org.nemours.gaitlab.repository;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public class AdminUserRepository {

    private final JdbcTemplate jdbcTemplate;

    public AdminUserRepository(
            JdbcTemplate jdbcTemplate
    ) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public List<AdminUserRow> findAllUsers() {

        return jdbcTemplate.query(
                """
                SELECT
                    userid,
                    fname,
                    lname,
                    facility,
                    access,
                    user_admin,
                    clin_type,
                    purpose,
                    active
                FROM public.users
                ORDER BY lname, fname, userid
                """,
                (rs, rowNum) ->
                        new AdminUserRow(
                                rs.getString("userid"),
                                rs.getString("fname"),
                                rs.getString("lname"),
                                rs.getString("facility"),
                                rs.getString("access"),
                                rs.getBoolean("user_admin"),
                                rs.getString("clin_type"),
                                rs.getString("purpose"),
                                rs.getBoolean("active")
                        )
        );
    }

    public record AdminUserRow(
            String userid,
            String fname,
            String lname,
            String facility,
            String access,
            boolean userAdmin,
            String clinType,
            String purpose,
            boolean active
    ) {
    }
}







package org.nemours.gaitlab.controller;

import org.nemours.gaitlab.repository.AdminUserRepository;
import org.nemours.gaitlab.repository.AdminUserRepository.AdminUserRow;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/api/admin/users")
public class AdminUserController {

    private final AdminUserRepository adminUserRepository;

    public AdminUserController(
            AdminUserRepository adminUserRepository
    ) {
        this.adminUserRepository =
                adminUserRepository;
    }

    @GetMapping
    public List<AdminUserRow> getUsers() {
        return adminUserRepository.findAllUsers();
    }
}
