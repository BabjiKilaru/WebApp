package org.nemours.gaitlab.controller;

import org.nemours.gaitlab.repository.UserRepository;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/api/admin/users")
public class AdminUserController {

    private final UserRepository userRepository;

    public AdminUserController(
            UserRepository userRepository
    ) {
        this.userRepository = userRepository;
    }

    @GetMapping
    public List<AdminUserResponse> getUsers() {

        return userRepository
                .findAdminUsers()
                .stream()
                .map(row ->
                        new AdminUserResponse(
                                stringValue(row[0]),
                                stringValue(row[1]),
                                stringValue(row[2]),
                                stringValue(row[3]),
                                stringValue(row[4]),
                                booleanValue(row[5]),
                                stringValue(row[6]),
                                stringValue(row[7]),
                                booleanValue(row[8])
                        )
                )
                .toList();
    }

    private String stringValue(
            Object value
    ) {
        return value == null
                ? null
                : value.toString();
    }

    private Boolean booleanValue(
            Object value
    ) {

        if (value == null) {
            return false;
        }

        if (value instanceof Boolean booleanValue) {
            return booleanValue;
        }

        return Boolean.parseBoolean(
                value.toString()
        );
    }

    public record AdminUserResponse(
            String userid,
            String fname,
            String lname,
            String facility,
            String access,
            Boolean userAdmin,
            String clinType,
            String purpose,
            Boolean active
    ) {
    }
}







package org.nemours.gaitlab.repository;

import org.nemours.gaitlab.model.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface UserRepository
        extends JpaRepository<User, String> {

    @Query(value = """
        SELECT
            userid,
            fname,
            lname,
            facility,
            access,
            user_admin,
            clintype,
            purpose,
            active
        FROM public.users
        ORDER BY lname, fname, userid
        """, nativeQuery = true)
    List<Object[]> findAdminUsers();
}
