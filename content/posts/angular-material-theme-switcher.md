---
title: "Angular Material theme switcher that doesn't suck"
date: 2020-03-01T18:28:20+01:00
draft: false
toc: true
---

# Abstract

For my project [TimeAtack](https://github.com/danielr1996/timeattack) I needed a way to switch between light and 
dark mode of an [Angular Material](https://material.angular.io/) Theme. This is documented in 
the [official theming guide](https://material.angular.io/guide/theming#defining-a-custom-theme) of Angular Material.

The guide mentions two methods of switching themes at runtime:

## Using CSS classes and OverlayContainer

This method uses additional css classes to distinguish themes and, for complexer component, 
a Service from the [CDK](https://material.angular.io/cdk/categories) called `OverlayContainer'`.
This method was not suitable for me because of 3 reasons:

- All styles must be loaded upfront which might not be an issue for 1 theme with dark/light variants but if you want to 
offer many themes your bundle size could be unnecessarily big.
- It just doesnt feel right to add that additional code to change css classes for some components and use 
`OverlayContainer` for other components
- It just didn't work: No matter I put my `mat-app-background` class, the background didn't 
change with the rest of component.

These were just my observations, maybe I missed something on the documentation. 
But after about 2 hours of googling I gave up on this approach.

## Pre compiling themes and referencing them with `<link rel="stylesheet">`

This second method seemed much simpler but there were some things missing or unclear in the official guide. 
That's why I'm writing this to safe you hours of googling.

The idea is pretty simple, you compile your theme from `scss` to `css` and reference it as any other stylesheet. 
In the next paragraph I'm going to explain what exactly needs to be done to make it work and where I struggled.

# How to do it right (at least in my opinion)
## Building the theme
To build your create a new `.scss` anywhere you like (I put mine under src/themes/theme-dark.scss) and put in the 
following content:
```.scss
@import '~@angular/material/theming';
@include mat-core();
$candy-app-primary: mat-palette($mat-indigo);
$candy-app-accent:  mat-palette($mat-pink, A200, A100, A400);
$candy-app-warn:    mat-palette($mat-red);
$candy-app-theme: mat-light-theme($candy-app-primary, $candy-app-accent, $candy-app-warn);
@include angular-material-theme($candy-app-theme);
```

I won't go into detail here because that's the usual way to build custom Angular Material Themes.
The guide mentions the following command to compile your theme to `css`:

```shell script
node-sass src/themes/theme-dark.scss src/assets/themes/theme-dark.css
```

Well seems easy, right? Not exactly. When I ran the command I got the following error:
```shell script
{
  "status": 1,
  "file": "C:/Users/Dani/Documents/Entwicklung/timeattack/src/themes/theme-dark.scss",
  "line": 1,
  "column": 1,
  "message": "File to import not found or unreadable: ~@angular/material/theming.",
  "formatted": "Error: File to import not found or unreadable: ~@angular/material/theming.\n        on line 1 of src/themes/theme-dark.scss\n>> @import '~@angular/material/theming';\r\n   ^\n"
}

```

But what does it mean? node-sass can't find the file `~@angular/material/theming`. That's what the error message says.
But why does it work if you put the code in your `styles.scss` and let it compile with `ng serve`. 
The reason is that the tilde (~) used to import from `node_modules` is not a builtin node-sass feature but realized with 
as an additional library called (node-sass-tilde-importer)[https://www.npmjs.com/package/node-sass-tilde-importer] that
the `ng serve` command uses internally. To use it with node-sass directly add the package to your project:
```shell script
npm install node-sass-tilde-importer --save-dev
```
and add the `--importer` flag to your node-sass command:
```shell script
node-sass src/themes/theme-dark.scss src/assets/themes/theme-dark.css --importer=node_modules/node-sass-tilde-importer
```
And now you have your compiled `css` files. 

You have to do this for all your themes (I have `theme-light.scss` and `theme-dark.scss`) which can get a bit tedious as
the number of themes grows. I'll show you how to automate that process later, but first let's see how to include 
the stylesheets and switch them at runtime.

## Including and switching the theme
To include your themes put the following tag in your `index.html` for your default theme.
```html
  <link id="theme" href="assets/themes/theme-light.css" rel="stylesheet ">
```

Now to switch the theme execute the following javascript:
```javascript
    // Get the <link> we inserted earlier by it's id
    let link = document.querySelector('#theme');
    
    // Replace the href attribute
    link.href = `assets/themes/theme-dark.css`;
```

This will already work and might get the idea how this work and write your own theme switcher and implement your own 
process of compiling all your themes. But if you want to follow along I'll explain how to automate the compilation with 
gulp and how to implement a simple dark/light mode switch.

## Automate the compilation with gulp
Install gulp and the required plugins
```shell script
npm install --save-dev gulp gulp-sass gulp-shell
```

Create the file `Gulpfile.js` at the root of your project and import all the required dependencies:
```javascript
const gulp = require('gulp');
const sass = require('gulp-sass');
const tilde = require('node-sass-tilde-importer');
const shell = require('gulp-shell')
```

Now add a task called themes to compile your `scss` files to `css`files:
```javascript
gulp.task('themes', function () {
  return gulp.src('src/themes/*.scss') //Take every scss file inside src/themes/
    .pipe(sass({importer: tilde})) // compile every scss file, this is essentially the same as the node-sass ... command we used earlier
    .pipe(gulp.dest('src/assets/themes')) //Write the css file to src/assets/themes/
    .pipe(shell(['ng build']))
});
```

To build your Angular app add a task called ng
```javascript
gulp.task('ng', shell.task('npx ng build --prod'));
```

And finally to build your app add a task `build` that calls `ng` after `themes`
```javascript
gulp.task('build', gulp.series(['themes', 'ng']));
``` 

The only draw back is that your themes won't get compiled if you just use `ng build` or `ng serve`.
But theme files don't change that often so for me it's that big of a problem.

## Building a theme switcher component
My theme switcher is just a regular Angular component that executed the code to switch the stylesheets 
that I mentioned earlier and stores the selected theme in the localstorage, so I won't go into detail here.


```typescript 
import {Component, OnInit} from '@angular/core';

@Component({
  selector: 'app-theme-switcher',
  template: `
    <button (click)="toggleDarkMode()" matTooltip="Toggle Darkmode"
            mat-icon-button
            aria-label="Toggle Darkmode">
      <mat-icon *ngIf="theme">brightness_4</mat-icon>
      <mat-icon *ngIf="!theme">brightness_5</mat-icon>
    </button>`,
  styleUrls: ['./theme-switcher.component.scss']
})
export class ThemeSwitcherComponent {
  public theme = localStorage.getItem('timeattack_theme');

  toggleDarkMode() {
    let link: any = document.querySelector('#theme');
    if (this.theme === 'dark') {
      this.theme = 'light';
      link.href = 'assets/themes/theme-light.css';
    } else {
      this.theme = 'dark';
      link.href = 'assets/themes/theme-dark.css';
    }
    localStorage.setItem('timeattack_theme', this.theme)
  }

}
```
To load the selected theme include the following javascript right after the `<link rel="stylesheet">` you added earlier.

```html
 <script>
    const theme = localStorage.getItem('timeattack_theme') || 'light';
    let link = document.querySelector('#theme');
    link.href = `assets/themes/theme-${theme}.css`;
  </script>
```

And that's it. Now you have a dark/light mode switch which can easily be extend to switch between an arbitraty number of different themes.

What's your opinion on this approach? Let me know in the comments.


# Sources
[stackoverflow.com: Replacing css file on the fly with javscript](https://stackoverflow.com/questions/19844545/replacing-css-file-on-the-fly-and-apply-the-new-style-to-the-page)

[npmjs.com: node-sass-tilde-importer](https://www.npmjs.com/package/node-sass-tilde-importer)

[material.angular.io: Theming Guide](https://material.angular.io/guide/theming#changing-styles-at-run-time)

[timeattack: Production Examples](https://github.com/danielr1996/timeattack)