# ASP.NET Core MVC Tutorial - CRUD 
By the end of this tutorial you'll have a fully functional ***TODO List***.
App name on this example will be **School**, modeled as a rooster of **Student** objects.
## *First things first...*
### - Environment:
```bash
neofetch               # Linux Mint 21.3 x86_64
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
using System.ComponentModel; //DisplayName and Range
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
### 6- Let's edit our ***Program.cs***:
Add these imports:
```
using School.Data;
using Microsoft.EntityFrameworkCore;
```
Find this comment:
```csharp
// Add services to the container.
```
Append these two. The *connection to the database* and the *runtime compiler* (it helps working on the front-end without having to re-compile)
```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
options.UseMySql(builder.Configuration.GetConnectionString("DefaultConnection"),
new MySqlServerVersion(new Version(8, 0, 35))));

builder.Services.AddRazorPages().AddRazorRuntimeCompilation();
```


### 7- Let's create and apply the **migrations**:
```bash
dotnet ef migrations add StudentsMigration
dotnet ef database update
```
### 8- Time to code the **Controller**:
```bash
touch Controllers/StudentController.cs
```

```csharp
namespace School.Controllers;
using Microsoft.AspNetCore.Mvc;
using School.Data;
using School.Models;

public class StudentController : Controller
{
  // Connecting the controller to the database
  private readonly Connection _db;

  public StudentController(Connection db)
  {
    _db = db;
  }

  // GET request retrieving all students from the database
  // and rendering them through the view "Student/Index" 
  public IActionResult Index() 
  {
    IEnumerable<Student> allStudents = _db.Students;
    return View(allStudents);
  }

  // GET request rendering the view "Student/Create"
  public IActionResult Create() //GET
  {
    return View();
  }

  // POST request adding a new entry,
  // and returning to "Index" if form data is valid
  [HttpPost]
  [ValidateAntiForgeryToken]
  public IActionResult Create(Student obj) //POST
  {
    if(ModelState.IsValid)
    {
      _db.Students.Add(obj);
      _db.SaveChanges();
      TempData["success"] = $"'{obj.FName}' entry created successfully!";
      return RedirectToAction("Index");
    }
    return View(obj);
  }

  // GET request retrieving an entry,
  // and rendering it through view "Student/Update"
  public IActionResult Update(int? id)
  {
    if (id==null || id==0)
    {
      return NotFound();
    }

    Student? alumni = _db.Students.Find(id);
    // these are also valid:
    // var alumni = _db.Students.FirstOrDefault(u=>u.Id==id);
    // var alumni = _db.Students.SingleOrDefault(u=>u.Id==id);

    if (alumni == null)
    {
      return NotFound();
    }
    return View(alumni);
  }

  // POST request updating selected entry
  // and returning to "Index" if form data is valid
  [HttpPost]
  [ValidateAntiForgeryToken]
  public IActionResult Update(Student obj) 
  {
    if(ModelState.IsValid)
    {
      _db.Students.Update(obj);
      _db.SaveChanges();
      TempData["success"] = $"Updated '{obj.FName}' successfully!";
      return RedirectToAction("Index");
    }
    return View(obj);
  }

  // GET request retrieving an entry,
  // and rendering it through view "Student/Delete"
  public IActionResult Delete(int? id) 
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

  // POST request deleting selected entry
  // and returning to "Index" if form data is valid
  // example below showcases use-case of "ActionName" annotation.
  // i.e.: endpoint "Remove" is overwritten as "Delete"
  [HttpPost, ActionName("Delete")]
  [ValidateAntiForgeryToken]
  public IActionResult Remove(Student obj) //POST
  {
    _db.Students.Remove(obj);
    _db.SaveChanges();
    TempData["success"] = $"Deleted '{obj.FName}' successfully!";
    return RedirectToAction("Index");
  }

}

```
The **ValidateAntiForgeryToken** is a security measure to prevent cross-site request forgery (CSRF) attacks.
The **TempData** helps us display a message right after an action is performed. We'll be using a partial view for that. See *step 10*.
### 9- Time to code the **Views**:
```bash
mkdir -p Views/Student
```
There will be one view per *GET* endpoint. Four in total:
This one below takes an ```IEnumerable<Student>``` object and renders each element inside a table row. 
``` bash
touch Views/Student/Index.cshtml
```
```html
@model IEnumerable<Student>
@{
  ViewData["Title"] = "Index";
}

<a asp-controller="Student" asp-action="Create"><i class="bi bi-plus-square"></i>New</a>
<table class="table table-bordered table-striped table-info" style="witdh:100%">
  <thead>
    <tr><th>Name</th><th>Order</th><th>Action</th></tr>
  </thead>
  <tbody>
      @foreach(var obj in Model)
      {
      <tr>
      <td>@obj.FName</td>
      <td>@obj.Age</td>
      <td>
          <a asp-controller="Student" asp-action="Update" asp-route-id="@obj.Id"> <i class="bi bi-pencil-square"></i>edit</a>
          <a asp-controller="Student" asp-action="Delete" asp-route-id="@obj.Id"> <i class="bi bi-trash"></i>delete </a>
      </td>
      </tr>
      }
  </tbody>
</table>
```
This view creates a `Student` object from form data, checks if the data is valid, and if so, inserts the entry into the database, sets a success message, and redirects to the index. Otherwise, it returns to the form with validation errors.
``` bash
touch Views/Student/Create.cshtml
```
```html
@model Student
<form method="post" asp-controller="Student" asp-action="Create">
      <label asp-for="FName"></label>
      <input asp-for="FName">
      <span asp-validation-for="FName"></span>  
      <label asp-for="Age"></label>
      <input asp-for="Age">
      <span asp-validation-for="Age"></span>  
      <button type="submit">create</button>
      <a asp-controller="Student" asp-action="Index">Back</a>
</form>
@section Scripts{
@{
<partial name="_ValidationScriptsPartial"/>
}
}
```
This view creates a `Student` object from form data and checks if the data is valid. If so, the entry in the database that matches the *Id* property is overwritten. In case there's no need to update a property (i.e.: *Age*), all you need is to add a hidden input like this: `<input type="hidden" asp-for="Age">`
``` bash
touch Views/Student/Update.cshtml
```
```html
@model Student
<form method="post" asp-controller="Student" asp-action="Update" asp-route-id="@Model.Id">
      <label asp-for="FName"></label>
      <input asp-for="FName">
      <span asp-validation-for="FName"></span>  
      <label asp-for="Age"></label>
      <input asp-for="Age">
      <span asp-validation-for="Age"></span>  
      <button type="submit">update</button>
      <a asp-controller="Student" asp-action="Index">Back</a>
</form>
@section Scripts{
@{
<partial name="_ValidationScriptsPartial"/>
}
}

```
This view deletes a `Student` object that matches the "@Model.Id"
``` bash
touch Views/Student/Delete.cshtml
```
```html
@model Student
<form method="post" asp-controller="Student" asp-action="Delete" asp-route-id="@Model.Id">
  <input type="hidden" asp-for="FName">
  <input type="hidden" asp-for="Age">
  <span>@Model.FName will be deleted.</span>
  <button type="submit">continue</button>
  <a asp-controller="Student" asp-action="Index">Back</a>
</form>
@section Scripts{
@{
<partial name="_ValidationScriptsPartial"/>
}
}
```
The **@model** directive declares the type of the model that the view expects (i.e.: from *controller*), **@Model** allows you to access properties of that model (i.e.: *@Model.Id*, *@Model.FName*, *@Model.Age*), and **@{ }** lets you write C# code directly in your Razor view.

At the bottom of each form (*Create*, *Update* & *Delete*), a "Scripts" section is created to include a partial view that performs client-side validation (this "Views/Shared/_ValidationScriptsPartial" partial-view is automatically generated when initiating the project).

### 10- Let's make it prettier!
Let's add **[Toastr](https://github.com/CodeSeven/toastr)** to have good looking notifications!
First, let's create a new partial view:
```bash
touch Views/Shared/_Notification.cshtml
```
The code below checks whether there is a value stored in *TempData* with the key "success"/"error" If such a value exists, it executes JavaScript code to display a success message using the **Toastr** library.
```html
@if(TempData["success"] != null)
{
  <script src="~/lib/jquery/dist/jquery.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/toastr.js/latest/toastr.min.js"></script>
  <script>
    toastr.success('@TempData["success"]');
  </script>
}

@if(TempData["error"] != null)
{
  <script src="~/lib/jquery/dist/jquery.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/toastr.js/latest/toastr.min.js"></script>
  <script>
    toastr.success('@TempData["error"]');
  </script>
}
```
Now let's edit the ***Views/Shared/_Layout.cshtml***:
Add **Toastr** and **Bootstrap-icons** css-stylesheet to the *head*:
```html
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.min.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/toastr.js/latest/toastr.min.css">
```
Find *@RenderBody()*. Right above it, add this *partial* tag:
```html
<partial name="_Notification">
```
Finally, add **Bootstrap-icons** JavaScript to *footer*:
```html
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/js/bootstrap.bundle.min.js" integrity="sha384-MrcW6ZMFYlzcLA8Nl+NtUVF0sA7MsXsP1UyJoMp4YLEuNSfAP+JcXn/tWtIaxVXM" crossorigin="anonymous"></script>
```


