# Foreign Key Relationships

Foreign Keys help us link data from two separate tables. In the example here, I'm creating a Meal. To ensure my meal is healthy, I want every meal to have exactly one source of protein. While I could let the user enter a protein as a string and store that, I don't want to give them that freedom. Instead, I want to provide a dropdown with a set list of proteins. So I'll need a protein table to store all the possible proteins:

| Id |   Title | CaloriesPerOunce |
|----|----------------------------|-----------------------------|
| 1  | Chicken | 35               |
| 2  | Beef    | 50               |
| 3  | Tofu    | 20               |

In Model code, this would look like:

    using System;
    using System.Collections.Generic;
    using System.ComponentModel.DataAnnotations;
    using System.ComponentModel.DataAnnotations.Schema;
    using System.Linq;
    using System.Threading.Tasks;
      
    namespace capstone.Models
    {
        public class Protein
        { 
            [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
            [Key]
            public int Id { get; set; }
            
            public string Title { get; set; }
            
            public int CaloriesPerOunce { get; set; }
        }
    }

Now for each meal, I want it to reference a protein that it'll use. This is where foreign keys come in!!

For example, if my Meal table looks like:

| Id |   Title | MeatId | Total Calories |
|----|----------------------------|-----------------------------|-----|
| 1  | Tofu Stir Fry | 3 | 150 |
| 2  | Chicken Salad | 1 | 200 |

That `MeatId` column is actually a foreign key or "reference" to the Meat table! So it's saying that Tofu Stir Fry uses `MeatId` 3, which, looking at the Meat table, is tofu!

If we have a foreign key relationship here, we'll also have some safety in our database. If I try to insert a new meal into the Meal table with a bad `MeatId` (one that doesn't exist in the Meat table), SQLite simply won't let me! So you'll have some built in safety if your users manage to bypass your frontend and send you bad data.

And the coolest part here is that in the C# code, you don't even have to play with Id's, all you see is that Meal has a Meat data member:

    using System;
    using System.Collections.Generic;
    using System.ComponentModel.DataAnnotations;
    using System.ComponentModel.DataAnnotations.Schema;
    using System.Linq;
    using System.Threading.Tasks;
      
    namespace capstone.Models
    {
        public class Meal
        {
            [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
            [Key]
            public int Id { get; set; }
            
            public string Title { get; set; }
            
            public int Calories { get; set; }
            
            // The actual Id the table is storing. This is the foregin key!
            public int MeatId { get; set; } 

            [ForeignKey("MeatId")] // <-- Needs to match the name of the ForeignKey property
            public Meat MeatItem { get; set; }
        }
    }

If you get this far, I'd try for a migration right now. When you open the migration file to check it (which you should do before you update your database!), you should look for the `Up()` function. This is the changes that are about to be applied to your database (`Down() would be the logic that gets run if you rollback this migration). Your `Up()` should look something like:

    migrationBuilder.CreateTable(
        name: "Proteins",
        columns: table => new
        {
            Id = table.Column<int>(nullable: false)
            .Annotation("Sqlite:Autoincrement", true),
            Title = table.Column<string>(nullable: true)
        },
        constraints: table =>
        {
            table.PrimaryKey("PK_Proteins", x => x.Id);
        });

    migrationBuilder.CreateTable(
        name: "Meals",
        columns: table => new
        {
            Id = table.Column<int>(nullable: false)
            .Annotation("Sqlite:Autoincrement", true),
            Title = table.Column<string>(nullable: true),
            MeatId = table.Column<int>(nullable: true),
            Calories = table.Column<int>(nullable: false)
        
        },
        constraints: table =>
        {
            table.PrimaryKey("PK_Meals", x => x.Id);
            table.ForeignKey(
            name: "FK_Meals_Meats_MeatId",
            column: x => x.MeatId,
            principalTable: "Meats",
            principalColumn: "Id",
            onDelete: ReferentialAction.Restrict);
        });

Notice how `MeatItem` isn't going to be part of the table at all. All the table holds is the foreign key `MeatId`, but our ORM will automagically transform that into a Meat item for us to use in C#!

For more on db relationships, see [the Microsoft docs](https://docs.microsoft.com/en-us/ef/core/modeling/relationships).
