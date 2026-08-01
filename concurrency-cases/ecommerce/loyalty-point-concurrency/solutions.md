# Thiết kế số dư điểm và sổ cái an toàn

## 1. Mục tiêu thiết kế

Giải pháp chính phải bảo đảm:

1. Số dư điểm không âm sau mọi giao dịch đã chốt.
2. Cộng và trừ đồng thời không ghi đè chênh lệch của nhau.
3. Mỗi thay đổi số dư thành công có đúng một bút toán chỉ thêm mới.
4. Bảng số dư luôn bằng tổng sổ cái.
5. Cùng mã lệnh và cùng nội dung chỉ được xử lý một lần.
6. Kết quả thành công hoặc thiếu điểm đều có thể phát lại.
7. Lỗi ở bất kỳ bước ghi nào hoàn tác toàn bộ giao dịch.

## 2. Lược đồ dữ liệu

```sql
CREATE TABLE loyalty_account (
    customer_id UUID PRIMARY KEY,
    points_balance BIGINT NOT NULL,
    lifetime_earned BIGINT NOT NULL DEFAULT 0,
    revision BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,

    CONSTRAINT ck_loyalty_account_balance
        CHECK (points_balance >= 0),
    CONSTRAINT ck_loyalty_account_lifetime_earned
        CHECK (lifetime_earned >= 0),
    CONSTRAINT ck_loyalty_account_revision
        CHECK (revision >= 0)
);

CREATE TABLE loyalty_command (
    customer_id UUID NOT NULL,
    command_id UUID NOT NULL,
    request_fingerprint CHAR(64) NOT NULL,
    operation VARCHAR(20) NOT NULL,
    requested_points BIGINT NOT NULL,
    order_id UUID,
    status VARCHAR(16) NOT NULL,
    outcome VARCHAR(32),
    balance_after BIGINT,
    account_sequence BIGINT,
    created_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,

    CONSTRAINT pk_loyalty_command
        PRIMARY KEY (customer_id, command_id),
    CONSTRAINT fk_loyalty_command_account
        FOREIGN KEY (customer_id)
        REFERENCES loyalty_account(customer_id),
    CONSTRAINT ck_loyalty_command_operation
        CHECK (operation IN ('EARN', 'REDEEM')),
    CONSTRAINT ck_loyalty_command_points
        CHECK (requested_points > 0),
    CONSTRAINT ck_loyalty_command_state
        CHECK (
            (
                status = 'PROCESSING'
                AND outcome IS NULL
                AND balance_after IS NULL
                AND account_sequence IS NULL
                AND completed_at IS NULL
            )
            OR
            (
                status = 'COMPLETED'
                AND outcome IN ('EARNED', 'REDEEMED')
                AND balance_after IS NOT NULL
                AND account_sequence IS NOT NULL
                AND completed_at IS NOT NULL
            )
            OR
            (
                status = 'COMPLETED'
                AND outcome = 'INSUFFICIENT_POINTS'
                AND operation = 'REDEEM'
                AND balance_after IS NULL
                AND account_sequence IS NULL
                AND completed_at IS NOT NULL
            )
        )
);

CREATE TABLE loyalty_ledger_entry (
    entry_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    command_id UUID NOT NULL,
    account_sequence BIGINT NOT NULL,
    entry_type VARCHAR(24) NOT NULL,
    points_delta BIGINT NOT NULL,
    balance_after BIGINT NOT NULL,
    order_id UUID,
    reverses_entry_id UUID,
    created_at TIMESTAMPTZ NOT NULL,

    CONSTRAINT fk_ledger_command
        FOREIGN KEY (customer_id, command_id)
        REFERENCES loyalty_command(customer_id, command_id),
    CONSTRAINT fk_ledger_reversed_entry
        FOREIGN KEY (reverses_entry_id)
        REFERENCES loyalty_ledger_entry(entry_id),
    CONSTRAINT uk_ledger_customer_command
        UNIQUE (customer_id, command_id),
    CONSTRAINT uk_ledger_customer_sequence
        UNIQUE (customer_id, account_sequence),
    CONSTRAINT uk_ledger_single_reversal
        UNIQUE (reverses_entry_id),
    CONSTRAINT ck_ledger_entry_type
        CHECK (
            entry_type IN (
                'OPENING',
                'EARN',
                'REDEEM',
                'REVERSAL',
                'ADJUSTMENT'
            )
        ),
    CONSTRAINT ck_ledger_delta_non_zero
        CHECK (points_delta <> 0),
    CONSTRAINT ck_ledger_balance_after
        CHECK (balance_after >= 0)
);

CREATE INDEX ix_ledger_customer_created
    ON loyalty_ledger_entry (customer_id, created_at, entry_id);
```

`revision` vừa đánh dấu phiên bản bảng chiếu vừa cung cấp thứ tự bút toán trong
một tài khoản. Mọi thay đổi số dư tăng `revision` trong chính câu SQL, rồi dùng
giá trị trả về làm `account_sequence`.

Ràng buộc `uk_ledger_customer_command` là lớp phòng thủ nếu mã ứng dụng cố chèn
bút toán hai lần. Quyền xử lý phải được chiếm ở `loyalty_command` trước khi thay
đổi số dư, không đợi tới lúc chèn sổ cái mới phát hiện trùng.

## 3. Bảo vệ tính chỉ thêm mới

Tài khoản chạy ứng dụng không được sửa hoặc xóa bút toán:

```sql
REVOKE UPDATE, DELETE
ON loyalty_ledger_entry
FROM app_runtime;

GRANT SELECT, INSERT
ON loyalty_ledger_entry
TO app_runtime;
```

Tài khoản migration tách riêng và được kiểm soát. Điều chỉnh nghiệp vụ dùng một
bút toán mới; không cấp quyền sửa lịch sử cho đường xử lý thông thường.

## 4. Lệnh nghiệp vụ

```java
public record SpendPointsCommand(
    UUID customerId,
    UUID commandId,
    UUID orderId,
    long points
) {
    public SpendPointsCommand {
        Objects.requireNonNull(customerId);
        Objects.requireNonNull(commandId);
        Objects.requireNonNull(orderId);
        if (points <= 0) {
            throw new IllegalArgumentException("points must be positive");
        }
    }
}

public record EarnPointsCommand(
    UUID customerId,
    UUID commandId,
    UUID orderId,
    long points
) {
    public EarnPointsCommand {
        Objects.requireNonNull(customerId);
        Objects.requireNonNull(commandId);
        Objects.requireNonNull(orderId);
        if (points <= 0) {
            throw new IllegalArgumentException("points must be positive");
        }
    }
}

public record LoyaltyResult(
    LoyaltyOutcome outcome,
    Long balanceAfter,
    Long accountSequence,
    boolean replayed
) {
}
```

Không nhận một số có dấu rồi suy ra cộng hay trừ. Hai kiểu lệnh tách riêng giúp
kiểm tra đầu vào và phân quyền rõ ràng. `customerId` phải đến từ tài khoản/đơn
hàng có thẩm quyền, không tin trường tùy ý do phía gọi gửi.

## 5. Dấu vân tay lệnh

```java
@Component
public class LoyaltyFingerprint {

    public String forSpend(SpendPointsCommand command) {
        return hash(String.join(
            "\n",
            "schema=v1",
            "operation=REDEEM",
            "customerId=" + command.customerId(),
            "commandId=" + command.commandId(),
            "orderId=" + command.orderId(),
            "points=" + command.points()
        ));
    }

    public String forEarn(EarnPointsCommand command) {
        return hash(String.join(
            "\n",
            "schema=v1",
            "operation=EARN",
            "customerId=" + command.customerId(),
            "commandId=" + command.commandId(),
            "orderId=" + command.orderId(),
            "points=" + command.points()
        ));
    }

    private String hash(String canonical) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            return HexFormat.of().formatHex(
                digest.digest(canonical.getBytes(StandardCharsets.UTF_8))
            );
        } catch (NoSuchAlgorithmException impossible) {
            throw new IllegalStateException("SHA-256 is unavailable", impossible);
        }
    }
}
```

Cùng `command_id` nhưng khác loại lệnh, số điểm hoặc đơn hàng phải bị từ chối,
không được phát lại một kết quả không liên quan.

## 6. Kho lưu mã lệnh

```java
public enum LoyaltyOperation {
    EARN,
    REDEEM
}
```

```java
@Repository
public class LoyaltyCommandStore {

    private final NamedParameterJdbcTemplate jdbc;

    public LoyaltyCommandStore(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public boolean tryClaim(
        UUID customerId,
        UUID commandId,
        String fingerprint,
        LoyaltyOperation operation,
        long points,
        UUID orderId
    ) {
        int changed = jdbc.update(
            """
            INSERT INTO loyalty_command (
                customer_id,
                command_id,
                request_fingerprint,
                operation,
                requested_points,
                order_id,
                status,
                created_at
            )
            VALUES (
                :customerId,
                :commandId,
                :fingerprint,
                :operation,
                :points,
                :orderId,
                'PROCESSING',
                CURRENT_TIMESTAMP
            )
            ON CONFLICT (customer_id, command_id) DO NOTHING
            """,
            new MapSqlParameterSource()
                .addValue("customerId", customerId)
                .addValue("commandId", commandId)
                .addValue("fingerprint", fingerprint)
                .addValue("operation", operation.name())
                .addValue("points", points)
                .addValue("orderId", orderId)
        );
        return changed == 1;
    }

    public StoredLoyaltyCommand find(
        UUID customerId,
        UUID commandId
    ) {
        return jdbc.queryForObject(
            """
            SELECT request_fingerprint,
                   operation,
                   requested_points,
                   order_id,
                   status,
                   outcome,
                   balance_after,
                   account_sequence
            FROM loyalty_command
            WHERE customer_id = :customerId
              AND command_id = :commandId
            """,
            Map.of(
                "customerId", customerId,
                "commandId", commandId
            ),
            (rs, rowNumber) -> new StoredLoyaltyCommand(
                rs.getString("request_fingerprint"),
                LoyaltyOperation.valueOf(rs.getString("operation")),
                rs.getLong("requested_points"),
                rs.getObject("order_id", UUID.class),
                LoyaltyCommandStatus.valueOf(rs.getString("status")),
                nullableOutcome(rs.getString("outcome")),
                rs.getObject("balance_after", Long.class),
                rs.getObject("account_sequence", Long.class)
            )
        );
    }

    public void completeSuccess(
        UUID customerId,
        UUID commandId,
        LoyaltyOutcome outcome,
        BalanceChange balance
    ) {
        int changed = jdbc.update(
            """
            UPDATE loyalty_command
            SET status = 'COMPLETED',
                outcome = :outcome,
                balance_after = :balanceAfter,
                account_sequence = :sequence,
                completed_at = CURRENT_TIMESTAMP
            WHERE customer_id = :customerId
              AND command_id = :commandId
              AND status = 'PROCESSING'
            """,
            new MapSqlParameterSource()
                .addValue("customerId", customerId)
                .addValue("commandId", commandId)
                .addValue("outcome", outcome.name())
                .addValue("balanceAfter", balance.pointsBalance())
                .addValue("sequence", balance.accountSequence())
        );
        requireOne(changed);
    }

    public void completeInsufficient(
        UUID customerId,
        UUID commandId
    ) {
        int changed = jdbc.update(
            """
            UPDATE loyalty_command
            SET status = 'COMPLETED',
                outcome = 'INSUFFICIENT_POINTS',
                completed_at = CURRENT_TIMESTAMP
            WHERE customer_id = :customerId
              AND command_id = :commandId
              AND status = 'PROCESSING'
              AND operation = 'REDEEM'
            """,
            Map.of(
                "customerId", customerId,
                "commandId", commandId
            )
        );
        requireOne(changed);
    }

    private LoyaltyOutcome nullableOutcome(String value) {
        return value == null
            ? null
            : LoyaltyOutcome.valueOf(value);
    }

    private void requireOne(int changed) {
        if (changed != 1) {
            throw new IllegalStateException(
                "Loyalty command was not completed exactly once"
            );
        }
    }
}
```

`ON CONFLICT DO NOTHING` không làm giao dịch rơi vào trạng thái lỗi như một vi
phạm duy nhất không được xử lý. Ở `READ COMMITTED`, bên thua có thể đọc bản ghi
đã chốt bằng câu `SELECT` kế tiếp.

## 7. Cập nhật số dư có thẩm quyền

```java
public record BalanceChange(
    long pointsBalance,
    long accountSequence
) {
}
```

```java
@Repository
public class LoyaltyAccountStore {

    private final NamedParameterJdbcTemplate jdbc;

    public LoyaltyAccountStore(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Optional<BalanceChange> debitIfEnough(
        UUID customerId,
        long points
    ) {
        List<BalanceChange> rows = jdbc.query(
            """
            UPDATE loyalty_account
            SET points_balance = points_balance - :points,
                revision = revision + 1,
                updated_at = CURRENT_TIMESTAMP
            WHERE customer_id = :customerId
              AND :points > 0
              AND points_balance >= :points
            RETURNING points_balance, revision
            """,
            Map.of(
                "customerId", customerId,
                "points", points
            ),
            (rs, rowNumber) -> new BalanceChange(
                rs.getLong("points_balance"),
                rs.getLong("revision")
            )
        );
        return rows.stream().findFirst();
    }

    public BalanceChange credit(
        UUID customerId,
        long points
    ) {
        return jdbc.queryForObject(
            """
            UPDATE loyalty_account
            SET points_balance = points_balance + :points,
                lifetime_earned = lifetime_earned + :points,
                revision = revision + 1,
                updated_at = CURRENT_TIMESTAMP
            WHERE customer_id = :customerId
              AND :points > 0
            RETURNING points_balance, revision
            """,
            Map.of(
                "customerId", customerId,
                "points", points
            ),
            (rs, rowNumber) -> new BalanceChange(
                rs.getLong("points_balance"),
                rs.getLong("revision")
            )
        );
    }
}
```

`loyalty_command` có khóa ngoại tới tài khoản, nên tài khoản phải tồn tại trước
khi chiếm lệnh. Không gộp tài khoản không tồn tại với thiếu điểm trong một kết
quả mơ hồ.

## 8. Thêm bút toán

```java
@Repository
public class LoyaltyLedgerStore {

    private final NamedParameterJdbcTemplate jdbc;

    public LoyaltyLedgerStore(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public UUID append(
        UUID customerId,
        UUID commandId,
        UUID orderId,
        LoyaltyEntryType entryType,
        long pointsDelta,
        BalanceChange balance
    ) {
        UUID entryId = UUID.randomUUID();

        jdbc.update(
            """
            INSERT INTO loyalty_ledger_entry (
                entry_id,
                customer_id,
                command_id,
                account_sequence,
                entry_type,
                points_delta,
                balance_after,
                order_id,
                created_at
            )
            VALUES (
                :entryId,
                :customerId,
                :commandId,
                :sequence,
                :entryType,
                :pointsDelta,
                :balanceAfter,
                :orderId,
                CURRENT_TIMESTAMP
            )
            """,
            new MapSqlParameterSource()
                .addValue("entryId", entryId)
                .addValue("customerId", customerId)
                .addValue("commandId", commandId)
                .addValue("sequence", balance.accountSequence())
                .addValue("entryType", entryType.name())
                .addValue("pointsDelta", pointsDelta)
                .addValue("balanceAfter", balance.pointsBalance())
                .addValue("orderId", orderId)
        );

        return entryId;
    }
}
```

Không có phương thức `update` hoặc `delete` cho sổ cái trong mã ứng dụng. Một
thao tác hoàn điểm dùng phương thức thêm bút toán riêng và ràng buộc
`reverses_entry_id`.

## 9. Giao dịch tiêu điểm

```java
@Service
public class LoyaltyTransaction {

    private final LoyaltyCommandStore commands;
    private final LoyaltyAccountStore accounts;
    private final LoyaltyLedgerStore ledger;

    public LoyaltyTransaction(
        LoyaltyCommandStore commands,
        LoyaltyAccountStore accounts,
        LoyaltyLedgerStore ledger
    ) {
        this.commands = commands;
        this.accounts = accounts;
        this.ledger = ledger;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public LoyaltyResult spend(
        SpendPointsCommand command,
        String fingerprint
    ) {
        boolean claimed = commands.tryClaim(
            command.customerId(),
            command.commandId(),
            fingerprint,
            LoyaltyOperation.REDEEM,
            command.points(),
            command.orderId()
        );

        if (!claimed) {
            return replaySpend(command, fingerprint);
        }

        Optional<BalanceChange> changed = accounts.debitIfEnough(
            command.customerId(),
            command.points()
        );

        if (changed.isEmpty()) {
            commands.completeInsufficient(
                command.customerId(),
                command.commandId()
            );
            return new LoyaltyResult(
                LoyaltyOutcome.INSUFFICIENT_POINTS,
                null,
                null,
                false
            );
        }

        BalanceChange balance = changed.orElseThrow();
        ledger.append(
            command.customerId(),
            command.commandId(),
            command.orderId(),
            LoyaltyEntryType.REDEEM,
            -command.points(),
            balance
        );
        commands.completeSuccess(
            command.customerId(),
            command.commandId(),
            LoyaltyOutcome.REDEEMED,
            balance
        );

        return new LoyaltyResult(
            LoyaltyOutcome.REDEEMED,
            balance.pointsBalance(),
            balance.accountSequence(),
            false
        );
    }

    private LoyaltyResult replaySpend(
        SpendPointsCommand command,
        String fingerprint
    ) {
        StoredLoyaltyCommand existing = commands.find(
            command.customerId(),
            command.commandId()
        );

        requireSameRequest(
            existing,
            fingerprint,
            LoyaltyOperation.REDEEM,
            command.points(),
            command.orderId()
        );

        if (existing.status() == LoyaltyCommandStatus.PROCESSING) {
            throw new LoyaltyCommandStillProcessingException();
        }

        return new LoyaltyResult(
            existing.outcome(),
            existing.balanceAfter(),
            existing.accountSequence(),
            true
        );
    }

    private void requireSameRequest(
        StoredLoyaltyCommand existing,
        String fingerprint,
        LoyaltyOperation operation,
        long points,
        UUID orderId
    ) {
        boolean same = existing.requestFingerprint().equals(fingerprint)
            && existing.operation() == operation
            && existing.requestedPoints() == points
            && Objects.equals(existing.orderId(), orderId);

        if (!same) {
            throw new LoyaltyCommandMismatchException();
        }
    }
}
```

Nhánh thiếu điểm không thay đổi số dư nên có thể chốt kết quả từ chối. Lỗi kỹ
thuật khi thêm bút toán hoặc hoàn tất lệnh phải đi ra khỏi Spring proxy để giao
dịch hoàn tác cả phép trừ.

## 10. Giao dịch cộng điểm

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public LoyaltyResult earn(
    EarnPointsCommand command,
    String fingerprint
) {
    boolean claimed = commands.tryClaim(
        command.customerId(),
        command.commandId(),
        fingerprint,
        LoyaltyOperation.EARN,
        command.points(),
        command.orderId()
    );

    if (!claimed) {
        return replayEarn(command, fingerprint);
    }

    BalanceChange balance = accounts.credit(
        command.customerId(),
        command.points()
    );
    ledger.append(
        command.customerId(),
        command.commandId(),
        command.orderId(),
        LoyaltyEntryType.EARN,
        command.points(),
        balance
    );
    commands.completeSuccess(
        command.customerId(),
        command.commandId(),
        LoyaltyOutcome.EARNED,
        balance
    );

    return new LoyaltyResult(
        LoyaltyOutcome.EARNED,
        balance.pointsBalance(),
        balance.accountSequence(),
        false
    );
}
```

`replayEarn` đối chiếu dấu vân tay, loại lệnh, số điểm và đơn hàng giống
`replaySpend`. Phần mã lặp có thể gom vào một hàm nhỏ; không tạo một khung xử lý
chung đến mức che mất sự khác nhau giữa phép cộng luôn hợp lệ và phép trừ có thể
bị từ chối.

## 11. Lớp ứng dụng và Spring proxy

```java
@Service
public class LoyaltyApplicationService {

    private final LoyaltyFingerprint fingerprints;
    private final LoyaltyTransaction transaction;

    public LoyaltyApplicationService(
        LoyaltyFingerprint fingerprints,
        LoyaltyTransaction transaction
    ) {
        this.fingerprints = fingerprints;
        this.transaction = transaction;
    }

    public LoyaltyResult spend(SpendPointsCommand command) {
        return transaction.spend(
            command,
            fingerprints.forSpend(command)
        );
    }

    public LoyaltyResult earn(EarnPointsCommand command) {
        return transaction.earn(
            command,
            fingerprints.forEarn(command)
        );
    }
}
```

Bean ngoài không mở giao dịch. Lời gọi sang `LoyaltyTransaction` đi qua Spring
proxy, nên ngoại lệ từ kho sổ cái hoặc kho lệnh thực sự làm hoàn tác giao dịch.

## 12. Vì sao các bất biến được bảo vệ

### Số dư không âm

Điều kiện và phép trừ nằm trong một câu SQL. Bên đến sau chờ khóa dòng rồi kiểm
tra lại số dư mới nhất. `CHECK` là lớp phòng thủ bổ sung.

### Không mất lần cộng hoặc trừ

Cả hai phép toán dùng `points_balance = points_balance + delta`, không ghi một
giá trị tuyệt đối đã tính từ ảnh chụp cũ.

### Sổ cái khớp số dư

Phép cập nhật số dư, chèn bút toán và hoàn tất lệnh cùng một giao dịch. Nếu một
bước thất bại, PostgreSQL hoàn tác tất cả.

### Không xử lý lệnh hai lần

Khóa chính của `loyalty_command` phân xử trước khi đụng số dư. Ràng buộc duy nhất
trên sổ cái là lớp phòng thủ thứ hai. Dấu vân tay ngăn cùng mã được dùng cho nội
dung khác.

## 13. Hành vi của bên thua

| Tranh chấp | Bên thua làm gì? |
| --- | --- |
| Hai mã khác nhau cùng tiêu | Chờ khóa số dư; sau đó thành công hoặc nhận `0` dòng |
| Cùng một mã lệnh | Chờ khóa duy nhất; sau đó phát lại kết quả |
| Bên giữ khóa hoàn tác | Có thể tiếp tục trên số dư/khóa lệnh như chưa có lần chốt |
| Hết thời gian chờ | Hoàn tác; không được báo thiếu điểm |
| Bế tắc | PostgreSQL hủy một giao dịch; thử lại toàn bộ bằng giao dịch mới |

## 14. Chính sách thử lại

- Không thử lại tự động kết quả `INSUFFICIENT_POINTS`.
- Có thể thử lại `55P03`, `40P01`, `40001` và lỗi kết nối phù hợp, với giới hạn
  số lần và thời gian tổng.
- Mỗi lần thử mở giao dịch mới và dùng cùng `command_id` cùng dấu vân tay.
- Mất phản hồi quanh `COMMIT` phải tra cứu trước; không tạo mã mới.
- Không giữ khóa số dư trong lúc chờ dịch vụ đơn hàng hoặc hệ thống thông điệp.
- Thử lại không thay thế việc thống nhất thứ tự khóa khi một giao dịch sửa nhiều
  tài khoản.

## 15. Hoàn điểm bằng bút toán bù

Một lần hoàn điểm tạo lệnh mới, cộng điểm và thêm bút toán `REVERSAL` trong cùng
giao dịch. `reverses_entry_id` liên kết bút toán gốc và có ràng buộc duy nhất để
ngăn hoàn hai lần.

Không dùng `DELETE` lịch sử cũ, không sửa `points_delta`, và không chỉ cộng số dư
mà thiếu bút toán bù.

## 16. Tích hợp với checkout và thông điệp

Nếu đơn hàng, điểm và mã checkout nằm trong cùng PostgreSQL, có thể đặt việc trừ
điểm trong cùng giao dịch tạo đơn, với một thứ tự khóa thống nhất. Giao dịch phải
ngắn và không gọi thanh toán từ xa.

Nếu dịch vụ điểm tách riêng, không thể dùng một `@Transactional` bao trùm hai cơ
sở dữ liệu. Checkout có thể cần giữ điểm, xác nhận, giải phóng và outbox/inbox.
Thông điệp có thể được giao lại nên cả phía gửi và phía nhận phải dùng mã lệnh ổn
định.

## 17. Các phương án thay thế

### `SELECT ... FOR UPDATE`

Khóa dòng trước khi đọc, kiểm tra trong Java, cập nhật số dư rồi thêm bút toán.
Cách này đúng khi giao dịch ngắn và quy tắc khó diễn đạt trong SQL. Nó giữ khóa
từ lúc đọc nên có thể tăng thời gian chờ.

### `@Version`

Hibernate sinh `UPDATE ... WHERE version = ?`; một bên nhận xung đột và phải chạy
lại toàn bộ giao dịch. Phù hợp khi tranh chấp hiếm. Dưới tải cao, lần thử lại có
thể khuếch đại tải lên tài khoản nóng.

### Chỉ dùng sổ cái và tính tổng mỗi lần

Lịch sử đầy đủ nhưng phép `SUM` không tự chiếm số dư. Muốn bảo vệ không âm vẫn
cần mức cô lập tuần tự hóa hoặc một dòng/quyền sở hữu khác. Đây không phải cách
đơn giản hơn bảng chiếu đồng bộ.

### Số dư bất đồng bộ từ sổ cái

Phù hợp cho màn hình chấp nhận dữ liệu trễ, nhưng không dùng một bảng chiếu trễ
để duyệt phép tiêu ngay lập tức. Quyết định cần nguồn số dư có thẩm quyền hoặc
cơ chế giữ hạn mức riêng.

### Khóa JVM hoặc khóa phân tán

Khóa JVM không bao phủ nhiều máy chủ. Khóa phân tán thêm lỗi thuê khóa và vẫn
không tạo lịch sử hay giao dịch nguyên tử với PostgreSQL.

## 18. So sánh định tính

| Cách | Tính đúng đắn | Hành vi tranh chấp | Tải và độ trễ | Độ phức tạp |
| --- | --- | --- | --- | --- |
| Trừ có điều kiện + sổ cái | Trực tiếp bảo vệ không âm và lịch sử | Chờ ngắn rồi nhận `1`/`0` dòng | Một dòng nóng trên mỗi khách hàng | Trung bình |
| `FOR UPDATE` | Đúng nếu khóa đủ dữ liệu | Chờ từ câu đọc | Khóa giữ lâu hơn | Trung bình |
| `@Version` | Đúng với thử lại chuẩn | Một bên lỗi phiên bản | Tăng lần thử khi tranh chấp | Trung bình |
| `SERIALIZABLE` + tính tổng | Có thể bảo vệ vị từ | Lỗi `40001`, thử lại toàn giao dịch | Quét lịch sử và xung đột vị từ | Cao |
| Bảng chiếu bất đồng bộ | Không phù hợp để duyệt số dư tức thời | Không chặn ở đường đọc | Đọc nhanh nhưng có độ trễ | Cao ở khâu vận hành |
| Khóa JVM | Không bảo vệ toàn hệ thống | Chỉ chờ cục bộ | Nhẹ nhưng sai khi nhiều máy chủ | Thấp nhưng không đạt yêu cầu |

## 19. Đối soát

```sql
SELECT a.customer_id,
       a.points_balance,
       COALESCE(sum(e.points_delta), 0) AS ledger_balance
FROM loyalty_account a
LEFT JOIN loyalty_ledger_entry e
  ON e.customer_id = a.customer_id
GROUP BY a.customer_id, a.points_balance
HAVING a.points_balance <> COALESCE(sum(e.points_delta), 0);
```

Truy vấn phải trả rỗng. Có thể kiểm tra thêm:

- lệnh `EARNED`/`REDEEMED` không có đúng một bút toán;
- bút toán không có lệnh tương ứng;
- `balance_after` không nối tiếp theo `account_sequence`;
- một bút toán bị đảo nhiều lần;
- lệnh `INSUFFICIENT_POINTS` lại có bút toán.

## 20. Chuyển đổi dữ liệu đang chạy

1. Đối soát số dư hiện tại với lịch sử đang có.
2. Chọn nguồn có thẩm quyền và tạo bút toán mở sổ/điều chỉnh có dấu vết.
3. Gán mã lệnh ổn định cho các đường ghi mới.
4. Loại bỏ hoặc hợp nhất bút toán trùng theo quyết định nghiệp vụ.
5. Thêm các ràng buộc duy nhất và `CHECK`.
6. Tách quyền migration khỏi quyền ứng dụng; thu hồi `UPDATE`/`DELETE` trên sổ.
7. Triển khai câu cập nhật có điều kiện và cập nhật chênh lệch.
8. Chạy đối soát liên tục sau khi phát hành.

Không xóa lịch sử hoặc sửa số dư âm thầm để migration “đi qua”. Việc điều chỉnh
điểm có thể ảnh hưởng đơn hàng và cần dấu vết kiểm toán.

## 21. Danh sách kiểm tra triển khai

- [ ] Số dư có `CHECK >= 0` và được trừ bằng câu SQL có điều kiện.
- [ ] Phép cộng dùng chênh lệch trên giá trị hiện tại trong PostgreSQL.
- [ ] Mã lệnh được chiếm trước khi thay đổi số dư.
- [ ] Dấu vân tay bao gồm loại lệnh, số điểm và tham chiếu đơn hàng.
- [ ] Số dư, bút toán và kết quả lệnh cùng một giao dịch.
- [ ] Kết quả thiếu điểm được lưu hoặc hợp đồng đánh giá lại được nêu rõ.
- [ ] Tài khoản ứng dụng không có quyền sửa/xóa sổ cái.
- [ ] Hoàn điểm dùng bút toán bù với ràng buộc chống hoàn hai lần.
- [ ] Không trộn thực thể JPA cũ với câu SQL trực tiếp.
- [ ] Không gọi mạng trong lúc giữ khóa số dư.
- [ ] Lỗi kỹ thuật và thiếu điểm được phân loại khác nhau.
- [ ] Có truy vấn đối soát và quy trình xử lý chênh lệch.
