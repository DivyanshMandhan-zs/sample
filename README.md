### @Transactional

Rocket now supports at @Transactional annotation which can used at method level. This annotation will invoke a transaction management in Rocket through which if an exception is encountered while making a query inside a method then all the preceding queries will also rollback.

> Note - This annotation is allowed only for classes annotated with `@Service` annotation. It is recommended to put this annotation on methods which are making DB calls.

Additionally, `@Transactional` now also supports nested transactions. This means that if a method annotated with `@Transactional` calls another method also annotated with `@Transactional`, each method operates within its own transaction scope. If an exception occurs after committing inner method DB calls, the changes made by the inner method will still be committed to the database. However, if an exception occurs in any subsequent part of the transaction, the root transaction will rollback, reverting any changes made by the inner method as well.
| Scenario |Outcome  |
|--|--|
| Outer method starts transaction. | Transaction begins. |
| Outer method calls inner method annotated with `@Transactional`. | Inner method starts its own transaction within the scope of the outer transaction. |
| Inner method performs database operations.| Changes made by the inner method are committed to the database and Outer method's transaction remains active.| 
| Exception occurs in the outer method. | Outer method's transaction is rolled back, reverting all changes made by both the outer and inner methods. |



![transaction](https://github.com/DivyanshMandhan-zs/demo-rocket/assets/125887675/844a122b-700a-46f3-a93f-11b2afd4c070)


### A Controller method looks like as follows.



```
@POST
@Path("/saveManagerAndUpdateEmployee/{managerName}/{employeeName}/{id}")
public void saveManagerAndUpdateEmployee(
        @PathParam("managerName") String managerName,
        @PathParam("employeeName") String employeeName,
        @PathParam("id") int employeeId) {
    ormService.saveManagerAndUpdateEmployee(managerName, employeeName, employeeId);
}
```

### A Service method looks like as follows.



```
@Transactional
public void saveManagerAndUpdateEmployee(
        String managerName, String employeeName, int employeeId) {
    try {
        if (managerName == null || managerName.isEmpty()) {
            throw new BadRequestException("Manager name cannot be null or empty");
        }
        if (employeeName == null || employeeName.isEmpty()) {
            throw new BadRequestException("Employee name cannot be null or empty");
        }
        Manager manager = new Manager();
        manager.setName(managerName);
        saveManager(manager);
        updateEmployeeNameById(employeeName, employeeId);
    } catch (RocketSQLException e) {
        throw new InternalServerException(e.getMessage(), e);
    }
}
```

Method **saveManager()**



```
@Transactional
public Integer saveManager(Manager manager) throws InternalServerException {
    try {
        return managerRepository.save(manager);
    } catch (RocketSQLException e) {
        throw new InternalServerException(e.getMessage(), e);
    }
}
```

Method **updateEmployeeNameById()**



```
@Transactional
public int updateEmployeeNameById(String name, int id) {
    try {
        if (employeeRepository.findById(id) != null) {
            return employeeRepository.updateEmployeeNameById(name, id);
        }
        throw new BadRequestException("Employee not found to update");
    } catch (RocketSQLException e) {
        throw new InternalServerException(e.getMessage(), e);
    }
}
```

### A Repository method looks like as follows.


```
@Repository
public interface EmployeeRepository extends ORMRepository<Employee, Integer> {

@Query("UPDATE Employee e SET e.name = ?1 WHERE e.id = ?2")
int updateEmployeeNameById(String name, int id);

}
```

If some Error occurs after **saveManger()** or **updateEmployeeNameById()** then the root transaction i.e for **saveManagerAndUpdateEmployee()** will also rollback.

Also now we are using Updated transactional methods to utilize Session instead of EntityManager for fetch queries.

1.  **Utilizing Hibernate's Session**: Instead of using the JPA EntityManager, we're using Hibernate's Session interface for transactional operations. The Session provides more fine-grained control over database operations and allows us to adhere more closely to Hibernate's best practices.
    
2.  **Consistency in Fetch Operations**: By aligning with Hibernate's recommended practices, we ensure consistency in fetch operations. This includes ensuring that transactions are properly managed, resources are released appropriately, and queries are executed efficiently.
    
3.  **Refactoring for Compatibility**: We've refactored affected code sections to maintain compatibility with Hibernate's session-based transaction handling. This may involve updating method signatures, modifying transactional annotations, or adjusting query execution logic to work seamlessly with the Session interface.
    
