# ASP.NET Core MVC Tutorial - CRUD 
## *First things first...*
### - Environment:
```bash
neofetch               # Linux Mint 21.2 x86_64
dotnet --list-runtimes # Microsoft.AspNetCore.App 7.0.15
dotnet --list-runtimes # Microsoft.NETCore.App 7.0.15
mysql --version        # mysql  Ver 8.0.35-0ubuntu0.22.04.1 for Linux on x86_64 ((Ubuntu))
```
### - Install the **.NET Software Development Kit** & **Entity Framework Core tools**:
```bash
apt install dotnet-host-7.0
apt install dotnet-sdk-7.0
dotnet tool install -g dotnet-ef  # dotnet ef --version
# dotnet tool update -g dotnet-ef --version 7.0.2

```
### - Setup **database**:
For most use-cases, an empty database & a single user with enough privileges will be enough.
- Installation
```bash
sudo apt install mysql-server
sudo mysql_secure_installation  #skip if you want
systemctl status mysql  #if inactive, do: systemctl start mysql
sudo mysql
```
- Create database and user
```sql
CREATE DATABASE databasename;
CREATE USER 'username'@'localhost' IDENTIFIED BY 'enterPasswordHere';
GRANT CREATE, DROP, ALTER, INSERT, SELECT, UPDATE, DELETE ON databasename.* to 'username'@'localhost';
```
New user can now access MySQL like this:
```
mysql -u username -p

USE DATABASE databasename; 
```
Or, with:
```
mysql -h localhost -u username -p databasename
```
## *Let's start the project!*
### 1- Initialize the project through the **.NET CLI**:
```bash
dotnet new mvc -n "School"
cd School #Root of the project
```
### 2- Add these **dependencies**:
```bash
dotnet add package Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation -v 7.0.0
dotnet add package Microsoft.EntityFrameworkCore -v 7.0.2
dotnet add package Microsoft.EntityFrameworkCore.Design -v 7.0.0
dotnet add package Pomelo.EntityFrameworkCore.MySql -v 7.0.0
```
### 3- Let's edit the app-settings:
Add this property to the ***appsettings.Development.json***
```json
"ConnectionStrings": {
      "DefaultConnection": "server=localhost;port=3306;database=databasename;user=username;password=enterPasswordHere;"
}
```
### 4- Time to code our **Model**:
```bash
touch Models/Student.cs
```
```csharp
namespace School.Models;
using System.ComponentModel.DataAnnotations; //[Key] & [Required]
using Component.Model; //[DisplayName] & [Range]
public class Student
{
[Key]
public int Id {get; set;}

[Required]
[DisplayName("First Name")]
public String FName {get; set;}

[Range(1,120, ErrorMessage="Valid ages: 1-120")]
public int Age {get; set;}
public DateTime MemberSince {get; set;} = DateTime.Now;
}
```

### 5- Now, let's code the **connection**:
```bash
mkdir Data
touch Data/Connection.cs
```
```csharp
namespace School.Data;
using Microsoft.EntityFrameworkCore;
using School.Models;

public class Connection: DbContext
{
  public Connection(DbContextOptions<Connection> options): base(options) {}

  public DbSet<Student> Students {get; set;}
}
```
### 6- Let's create and apply the **migrations**:
```bash
dotnet ef migrations add StudentsMigration
dotnet ef database update
```
### 7- Time to code the **Controller**:
```bash
touch Controllers/Student.cs
```
```csharp
namespace School.Controllers;
using Microsoft.AspNetCore.Mvc;
using Student.Data;
using Student.Models;

public class StudentController : Controller
{
  private readonly Connection _db;

  public StudentController(Connection db)
  {
    _db = db;
  }

  public IActionResult Index() //GET
  {
    IEnumerable<Student> allStudents = _db.Students;
    return View(allStudents);
  }

  public IActionResult Create() //GET
  {
    return View();
  }

  [HttpPost]
  [ValidateAntiForgeryToken]
  public IActionResult Create(Student obj) //POST
  {
    if(ModelState.IsValid)
    {
      _db.Students.Add(obj);
      _db.SaveChanges();
      TempData["success"] = $"'{obj.Name}' entry created successfully!";
      return RedirectToAction("Index");
    }
    return View(obj);
  }

  public IActionResult Update(int? id) //GET
  {
    if (id==null || id==0)
    {
      return NotFound();
    }

    Student? alumni = _db.Students.Find(id);
    // var alumni = _db.Students.FirstOrDefault(u=>u.Id==id);
    // var alumni = _db.Students.SingleOrDefault(u=>u.Id==id);
    if (alumni == null)
    {
      return NotFound();
    }
    return View(alumni);
  }

  [HttpPost, ActionName("Update")]
  [ValidateAntiForgeryToken]
  public IActionResult Change(Category obj) //POST
  {
    if(ModelState.IsValid)
    {
      _db.Students.Update(obj);
      _db.SaveChanges();
      TempData["success"] = $"Updated '{obj.Name}' successfully!";
      return RedirectToAction("Index");
    }
    return View(obj);
  }

  public IActionResult Delete(int? id) // GET
  {
    if (id == null || id == 0)
    {
      return NotFound();
    }
    var alumni = _db.Students.Find(id);

    if (alumni == null)
    {
      return NotFound();
    }

    return View(alumni);
  }

  [HttpPost, ActionName("Delete")]
  [ValidateAntiForgeryToken]
  public IActionResult Remove(Category obj) //POST
  {
    _db.Students.Remove(obj);
    _db.SaveChanges();
    TempData["success"] = $"Deleted '{obj.Name}' successfully!";
    return RedirectToAction("Index");
  }

}
```



