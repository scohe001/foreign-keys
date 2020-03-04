# Foreign Key Relationships

### Who will this be useful for?

This guide will be useful for you if you...

- Are still in the process of setting up your backend (database and models) for your Capstone
- You want the user to be able to enter items that will then later be used in a dropdown and stored as part of another object
- You have a one-to-many relationship (ie: in the example below, one Meal has many Ingredients)

For more on dotnet db relationships, you can also see the [the Microsoft docs](https://docs.microsoft.com/en-us/ef/core/modeling/relationships) (click the "Data Annotations" tab on examples where applicable to see how you should modify your Model code).

## Basic Foreign Key relationship

Foreign Keys help us link data from two separate tables. In my example capstone, I'm building meals. To ensure a meal is healthy, I want every meal to have exactly one source of protein. However, everyone uses different things as protein sources, so I want the user to be able to enter in their own protein sources and then when building a meal, choose from a dropdown of all of the proteins entered.

I could just have the user enter a string for the protein when creating a meal, but I like the idea of having a dropdown for them to pick from (and this also lets me store things on the protein like CaloriesPerOunce, which I can display alongside the proteins in the dropdown!).

So I'll need a protein table to store all the possible proteins:

| Id |   Title | CaloriesPerOunce |
|----|---------|------------------|
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
|----|---------|--------|----------------|
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

If you get this far, I'd try for a migration right now. When you open the migration file to check it (which you should do before you update your database!), you should look for the `Up()` function. This is the changes that are about to be applied to your database (`Down()` would be the logic that gets run if you rollback this migration). Your `Up()` should look something like:

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

## Inverse Property relationship

Having proteins is nice, but the way things are setup, each meal will only have one protein. What if I'm making a Pizza-Builder that needs to add a bunch of toppings?

Again, we'll need a Topping table (similar to the Protein table from before) that the user can populate by creating their own topping types:

| Id |   Title | CaloriesPerOunce |
|----|---------|------------------|
| 1  | Olives  | 35               |
| 2  | Mushroom | 50               |
| 3  | Pineapple | 20               |
| 3  | Chicken   | 70               |
| 3  | Pepperoni | 9999999      |

But how can we link this up with the Pizza table? If I have a `ToppingId` column on Pizza, then I can only hold one topping? How can we accomplish this?

**We'll actually need one more table to act as a linker!**


