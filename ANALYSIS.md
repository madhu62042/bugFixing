# T1 Analysis - LoanAccountService.getOverdueLoans()

## Root Cause

1. Result list was initialized as null.
   - Calling result.add(account) caused NullPointerException.
   - Fixed by initializing result with ArrayList.

2. dueDate can be null for restructured accounts.
   - Calling before() on null dueDate caused NullPointerException.
   - Added null check before date comparison.

3. Accounts with zero outstanding balance should not be returned.
   - Kept validation to return only accounts having outstanding balance greater than zero.

## Fix Applied

Initialize result to new ArrayList<>(), add a null check on getDueDate(), and replace > 0 with an epsilon-based comparison. 

# Code

public List<LoanAccount> getOverdueLoans(List<LoanAccount> accounts) {
   List<LoanAccount> result = new ArrayList<>();
   final double EPSILON = 0.0001;
   for (LoanAccount account : accounts) {
   if (account.getDueDate() != null && account.getDueDate().before(new Date())) {
   if (account.getOutstandingBalance() > EPSILON) {
                result.add(account);
            }
        }
    }
    return result;
}


# T2 Analysis - ConcurrentModificationException

## Root Cause

ConcurrentModificationException occurs when a collection is structurally modified
while it is being iterated using an Iterator or enhanced for loop.

## Possible Trigger Pattern

for (Transaction t : transactions) {
    if (!isValid(t)) {
        transactions.remove(t);   // structural modification during iteration
    }
}

The list modification changes the collection modification count, causing iterator
validation failure.

## Fix Applied
transactions.removeIf(t -> !isValid(t));




# T3 Analysis - Thread Safety

## Root Cause

processedCount++ is not atomic. It is actually three separate steps — read the current value, increment it, write it back. Under the 10-thread pool, two threads can read the same value before either writes its result back, so one thread's increment is silently lost. This is a classic race condition, and explains why the final count is inconsistent and frequently lower than the number of records actually processed.



## Fix Applied

 Replaced the plain int processedCount field with AtomicInteger, and replaced processedCount++ with processedCount.incrementAndGet(), which performs the increment  as a single atomic operation. getProcessedCount() now returns processedCount.get().
 Thread pool and processing logic were unchanged.

# Code

public class BankStatementBatchProcessor {
    private AtomicInteger processedCount = new AtomicInteger(0);

    public void process(List<StatementRecord> records) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(10);

        for (StatementRecord record : records) {
            executor.submit(() -> {
                processRecord(record);
                processedCount.incrementAndGet(); 
            });
        }
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.MINUTES);
    }

    public int getProcessedCount() {
        return processedCount.get(); 
    }
}

# T4 Analysis - Connection Leak

## Root Cause

Connection, PreparedStatement and ResultSet were created but never closed.

Because connections were not returned to the connection pool, continuous requests
caused connection pool exhaustion and application hanging.

## Fix Applied

-Implemented try-with-resources.

Resources are automatically closed in the correct order:

ResultSet → PreparedStatement → Connection

SQL query and mapRow() logic were not modified.

Wrapped all three resources in try-with-resources. Java closes try-with-resources blocks in reverse declaration order automatically, so nesting the ResultSet's try block inside the Connection/PreparedStatement try block produces the required closure order: ResultSet → PreparedStatement → Connection, with no manual finally block needed. Query logic and mapRow() are unchanged.

# Code

public class ReportDAO {
    private DataSource dataSource;
    public List<ReportEntry> fetchMonthlyReport(String accountId,
                                                  int month, int year)
                                                  throws SQLException {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "SELECT * FROM report_entries " +
                 "WHERE account_id = ? AND MONTH(entry_date) = ? " +
                 "AND YEAR(entry_date) = ?")) {
            ps.setString(1, accountId);
            ps.setInt(2, month);
            ps.setInt(3, year);
            try (ResultSet rs = ps.executeQuery()) {
                List<ReportEntry> entries = new ArrayList<>();
                while (rs.next()) {
                    entries.add(mapRow(rs));
                }
                return entries;
            }
        }
    }
}

# T5 — Exception Handling in DocumentValidator

## Root Cause

e.printStackTrace() flooding logs — replaced with logger.warn(...), logging only the message (no full stack trace), since these are expected validation failures (null document, empty content), not unexpected system errors.

return null on failure — replaced with ValidationResult.invalid(message), a proper result object, so callers no longer need defensive null checks and cannot NPE on the returned value.

Unsafe r.isValid() call in validateBatch() — this is resolved as a direct consequence of fixing issue 2: since validate() can no longer return null, calling r.isValid() is now safe without an additional check.

Silent exception swallowing

- The catch block in validateBatch() was empty.
- Exceptions were completely ignored, making production debugging difficult.
- Added proper logging for unexpected exceptions.

## Fix Applied

- Added SLF4J logging instead of printStackTrace().
- Differentiated expected validation failures from unexpected runtime errors.
- Prevented NullPointerException by validating returned results.
- Added logging for swallowed exceptions.
- Existing runValidationRules() and saveResult() logic were unchanged.
