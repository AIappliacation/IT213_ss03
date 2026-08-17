Bài 4: Sáng tạo — Module ETL Resume Parser
1. Kiến trúc tổng thể

Module được thiết kế theo mô hình:

                         RIKKEI ACADEMY HR
                                │
                                │ CV văn bản thô
                                ▼
                    ┌───────────────────────┐
                    │   CandidateETLService │
                    │                       │
                    │ processResume()       │
                    └───────────┬───────────┘
                                │
                         EXTRACT
                                │
                                ▼
                    ┌───────────────────────┐
                    │       ChatModel       │
                    │         + LLM         │
                    └───────────┬───────────┘
                                │
                         JSON có cấu trúc
                                │
                                ▼
                    ┌───────────────────────┐
                    │ BeanOutputConverter  │
                    │                       │
                    │ JSON → Java Record    │
                    └───────────┬───────────┘
                                │
                         TRANSFORM
                                │
                                ▼
                    ┌───────────────────────┐
                    │ CandidateExtraction   │
                    │                       │
                    │ fullName              │
                    │ phone                 │
                    │ email                 │
                    │ skills                │
                    │ yearsExperience       │
                    └───────────┬───────────┘
                                │
                          VALIDATION
                                │
                    ┌───────────▼───────────┐
                    │ Kiểm tra nghiệp vụ    │
                    │                       │
                    │ ✓ Tên không rỗng      │
                    │ ✓ Email hợp lệ        │
                    │ ✓ Experience >= 0     │
                    └───────────┬───────────┘
                                │
                          Mapping Entity
                                │
                                ▼
                    ┌───────────────────────┐
                    │       Candidate       │
                    │        @Entity        │
                    └───────────┬───────────┘
                                │
                              LOAD
                                │
                                ▼
                    ┌───────────────────────┐
                    │ CandidateRepository   │
                    │     JpaRepository     │
                    └───────────┬───────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │      SQL Database     │
                    │                       │
                    │      candidates       │
                    └───────────────────────┘

Có thể tóm tắt:

CV Raw
  ↓
ChatModel + Prompt
  ↓
LLM
  ↓
JSON
  ↓
BeanOutputConverter
  ↓
CandidateExtraction
  ↓
Validation
  ↓
Candidate Entity
  ↓
CandidateRepository.save()
  ↓
SQL Database
2. Java Record CandidateExtraction

Record là DTO trung gian nhận dữ liệu từ LLM.

package com.rikkei.hr.dto;


import java.util.List;


public record CandidateExtraction(
        String fullName,
        String phone,
        String email,
        List<String> skills,
        int yearsExperience
) {
}

Ví dụ LLM phải trả về:

{
  "fullName": "Nguyen Van An",
  "phone": "0901234567",
  "email": "an.nguyen@gmail.com",
  "skills": [
    "Java",
    "Spring Boot",
    "MySQL",
    "Docker"
  ],
  "yearsExperience": 3
}

Sau khi BeanOutputConverter xử lý:

JSON
  ↓
BeanOutputConverter
  ↓
CandidateExtraction
3. JPA Entity Candidate

Entity đại diện cho dữ liệu được lưu xuống database.

package com.rikkei.hr.entity;


import jakarta.persistence.*;


import java.util.ArrayList;
import java.util.List;


@Entity
@Table(name = "candidates")
public class Candidate {


    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;


    @Column(nullable = false)
    private String fullName;


    private String phone;


    @Column(nullable = false, unique = true)
    private String email;


    @ElementCollection
    @CollectionTable(
            name = "candidate_skills",
            joinColumns = @JoinColumn(name = "candidate_id")
    )
    @Column(name = "skill")
    private List<String> skills = new ArrayList<>();


    @Column(nullable = false)
    private int yearsExperience;


    public Candidate() {
    }


    public Candidate(
            String fullName,
            String phone,
            String email,
            List<String> skills,
            int yearsExperience
    ) {
        this.fullName = fullName;
        this.phone = phone;
        this.email = email;
        this.skills = skills;
        this.yearsExperience = yearsExperience;
    }


    public Long getId() {
        return id;
    }


    public String getFullName() {
        return fullName;
    }


    public String getPhone() {
        return phone;
    }


    public String getEmail() {
        return email;
    }


    public List<String> getSkills() {
        return skills;
    }


    public int getYearsExperience() {
        return yearsExperience;
    }


    public void setFullName(String fullName) {
        this.fullName = fullName;
    }


    public void setPhone(String phone) {
        this.phone = phone;
    }


    public void setEmail(String email) {
        this.email = email;
    }


    public void setSkills(List<String> skills) {
        this.skills = skills;
    }


    public void setYearsExperience(int yearsExperience) {
        this.yearsExperience = yearsExperience;
    }
}
Quan hệ với skills

Ở đây sử dụng:

@ElementCollection

vì skills chỉ là danh sách các chuỗi, chưa cần tạo Entity Skill riêng.

Database có thể có:

candidates
--------------------------------
id
full_name
phone
email
years_experience

và:

candidate_skills
--------------------------------
candidate_id
skill

Ví dụ:

candidates


1 | Nguyen Van An | 0901234567 | an@gmail.com | 3




candidate_skills


1 | Java
1 | Spring Boot
1 | MySQL
1 | Docker
4. CandidateRepository
package com.rikkei.hr.repository;


import com.rikkei.hr.entity.Candidate;
import org.springframework.data.jpa.repository.JpaRepository;


import java.util.Optional;


public interface CandidateRepository
        extends JpaRepository<Candidate, Long> {


    Optional<Candidate> findByEmail(String email);


    boolean existsByEmail(String email);
}

Việc thêm:

boolean existsByEmail(String email);

giúp kiểm tra ứng viên đã tồn tại trước khi insert.

5. CandidateETLService

Đây là thành phần quan trọng nhất của module.

package com.rikkei.hr.service;
    }


    private void validateExtraction(
            CandidateExtraction extraction
    ) {


        // Validation 1: họ tên không được trống
        if (extraction.fullName() == null
                || extraction.fullName().isBlank()) {


            throw new IllegalArgumentException(
                    "Candidate full name must not be empty"
            );
        }


        // Validation 2: email hợp lệ
        if (extraction.email() == null
                || !EMAIL_PATTERN
                .matcher(extraction.email())
                .matches()) {


            throw new IllegalArgumentException(
                    "Candidate email is invalid"
            );
        }


        // Validation 3: số năm kinh nghiệm >= 0
        if (extraction.yearsExperience() < 0) {


            throw new IllegalArgumentException(
                    "Years of experience must be >= 0"
            );
        }


        // Validation 4: skills không null
        if (extraction.skills() == null) {


            throw new IllegalArgumentException(
                    "Skills must not be null"
            );
        }
    }


    @Transactional
    protected Candidate saveCandidate(
            CandidateExtraction extraction
    ) {


        if (candidateRepository
                .existsByEmail(extraction.email())) {


            throw new IllegalArgumentException(
                    "Candidate email already exists"
            );
        }


        Candidate candidate = new Candidate(
                extraction.fullName(),
                extraction.phone(),
                extraction.email(),
                List.copyOf(extraction.skills()),
                extraction.yearsExperience()
        );


        return candidateRepository.save(candidate);
    }
}
6. Tại sao tách saveCandidate() thành một transaction riêng?

Đây là điểm rất quan trọng trong yêu cầu của bài.

Không nên giữ transaction database trong suốt thời gian gọi LLM.

Ví dụ cách không tối ưu:

@Transactional
public Candidate processResume(String resumeText) {


    // Gọi LLM
    String json = chatModel.call(prompt);


    // Validation
    ...


    // Save DB
    candidateRepository.save(candidate);


    return candidate;
}

Luồng:

BEGIN TRANSACTION
       │
       ▼
   Gọi LLM
       │
       │ 15 giây
       │
       ▼
   LLM Response
       │
       ▼
   Validation
       │
       ▼
   INSERT DB
       │
       ▼
COMMIT

Trong thời gian LLM đang xử lý, transaction có thể đã được mở và tùy vào cách ORM/transaction manager tương tác với persistence context, tài nguyên DB có thể bị giữ lâu hơn cần thiết.

Đặc biệt nếu hệ thống có:

100 request đồng thời
        ↓
100 transaction
        ↓
100 request gọi LLM
        ↓
connection pool bị áp lực

Điều này rất nguy hiểm khi LLM mất 15–20 giây.

7. Trade-off: LLM bên trong @Transactional
Cách 1 — Gọi LLM bên trong Transaction
@Transactional
public Candidate processResume(String resumeText) {


    CandidateExtraction data =
            callLLM(resumeText);


    validate(data);


    return repository.save(...);
}
Ưu điểm

1. Logic đơn giản

Toàn bộ workflow nằm trong một phương thức.

LLM
 ↓
Validation
 ↓
DB

2. Dễ hiểu về mặt transaction

Nếu phần database bị lỗi:

save()
 ↓
Exception
 ↓
ROLLBACK

thì các thao tác DB trong transaction được rollback.

Nhược điểm

1. Transaction có thể tồn tại quá lâu

LLM có thể mất:

15–20 giây

Trong khi thao tác SQL thực tế có thể chỉ mất:

10–100 ms

Không nên để tài nguyên transaction/database bị gắn với một network call chậm.

2. Tăng áp lực connection pool

Ví dụ:

Connection Pool = 10

Nếu nhiều transaction giữ connection trong lúc chờ LLM:

Request 1 → LLM 20s → Connection
Request 2 → LLM 20s → Connection
Request 3 → LLM 20s → Connection
...
Request 10 → LLM 20s → Connection

Request thứ 11 có thể phải chờ connection.

3. Rollback không thể rollback LLM

Đây là điểm cực kỳ quan trọng.

Giả sử:

BEGIN TRANSACTION
      ↓
Gọi LLM thành công
      ↓
LLM đã xử lý request
      ↓
INSERT DATABASE
      ↓
DATABASE ERROR
      ↓
ROLLBACK

Database rollback được.

Nhưng:

LLM

không rollback được theo transaction SQL.

LLM là một external service.