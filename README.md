mongoose-tsgen
==============

An out-of-the-box Typescript interface generator for Mongoose.

[![oclif](https://img.shields.io/badge/cli-oclif-brightgreen.svg)](https://oclif.io)
[![Version](https://img.shields.io/npm/v/mongoose-tsgen.svg)](https://npmjs.org/package/mongoose-tsgen)
[![npm](https://img.shields.io/npm/dt/mongoose-tsgen)](https://www.npmjs.com/package/mongoose-tsgen)
[![License](https://img.shields.io/npm/l/mongoose-tsgen.svg)](https://github.com/Bounced-Inc/mongoose-tsgen/blob/master/package.json)
<!-- [![Downloads/week](https://img.shields.io/npm/dw/mongoose-tsgen.svg)](https://npmjs.org/package/mongoose-tsgen) -->

<!-- toc -->
* [Features](#features)
* [Compatibility](#compatibility)
* [Installation](#installation)
* [Usage](#usage)
* [Example](#example)
<!-- tocstop -->

# Features

- [x] Automatically generate an `index.d.ts` file containing all your Mongoose Schemas as Typescript interfaces
- [x] Works out of the box, don't need to rewrite your schemas
- [x] Creates an interfaces for each Mongoose document, model and all subdocuments
- [x] Add custom interfaces (i.e. a subset of a document for use by a client)
- [x] Multiple search patterns and import strategies to require minimal input and configuration

# Compatibility

- [x] All Mongoose types and arrays
- [x] Virtual properties
- [x] Both Typescript and Javascript schema files (if using Javascript, you will want to convert to Typescript upon generating the `index.d.ts` file)
- [x] Typescript path aliases
- [x] Mongoose method and static functions - These could be improved, they currently get typed as `Function` without parameter and return types
- [ ] Support for `Model.Create`. Currently `new Model` must be used.
- [ ] Support for setting subdocument properties without casting to `any`. When setting a subdocument array property, you need to cast to `any` as Typescript expects the array to contain additional subdocument properties (ie `user.friends = [{ uid, name }]` should be written as `user.friends = [{ uid, name }] as any`).

Would love any help with the last few listed features above.

# Installation
<!-- usage -->
```sh-session
$ npm install -D mongoose-tsgen
$ npx mtgen --help # print usage
```
<!-- usagestop -->
# Usage
<!-- commands -->

## `mtgen [ROOT_PATH]`

Generate an index.d.ts file containing Mongoose Schema interfaces.

```
USAGE
  $ mtgen [ROOT_PATH - default = "."]

OPTIONS
  -d, --dry-run          print output rather than writing to file
  -f, --fresh            fresh run, ignoring previously generated custom interfaces
  -h, --help             show CLI help
  -j, --js               search for Mongoose schemas in Javascript files rather than in Typescript files
  -o, --output=output    [default: ./src/types/mongoose] path of output index.d.ts file
  -p, --project=project  [default: ./] path of tsconfig.json or its root folder
```

All sub-directories of `ROOT_PATH` will be searched for a `/models/` folder. If such folder contains an `index.{j|t}s` file, all Mongoose models are expected to be exported from here. If such file does not exist, all `*.{t|j}s` in this folder are expected to export a Mongoose model.

<i>NOTE: --output requires a folder path or a file path ending in `index.d.ts`. If the path does not exist, it will be created.</i>

_See code: [src/commands/run.ts](https://github.com/Bounced-Inc/mongoose-tsgen/blob/master/src/commands/run.ts)_
<!-- commandsstop -->

# Example

### ./src/models/user.ts

```typescript
// NOTE: you will need to import these types after your first ever run of the CLI
// See the 'Initializing Schemas' section
import mongoose, { IUser, IUserModel } from "mongoose";
const { Schema } = mongoose;

const UserSchema = new Schema({
  email: {
    type: String,
    required: true
  },
  firstName: {
    type: String,
    required: true
  },
  lastName: {
    type: String,
    required: true
  },
  metadata: Schema.Types.Mixed,
  friends: [
    {
      uid: {
        type: Schema.Types.ObjectId,
        ref: "User",
        required: true
      },
      nickname: String
    }
  ],
  city: {
    coordinates: {
      type: [Number],
      index: "2dsphere"
    }
  }
});

// NOTE: `this: IUser` and `this: IUserModel` is to tell TS the type of `this' value using the "fake this" feature
// you will need to add these in after your first ever run of the CLI

UserSchema.virtual("name").get(function(this: IUser) { return `${this.firstName} ${this.lastName}` });

// method functions
UserSchema.methods = {
  isMetadataString(this: IUser) { return typeof this.metadata === "string"; }
}

// static functions
UserSchema.statics = {
  // friendUids could also use the type `ObjectId[]` here
  async getFriends(this: IUserModel, friendUids: IUser["_id"][]) {
    return await this.aggregate([ { $match: { _id: { $in: friendUids } } } ]);
  }
}

export const User: IUserModel = mongoose.model<IUser, IUserModel>("User", UserSchema);
export default User;
```

### generate interfaces

```bash
$ mtgen
```

### generated interface file ./src/types/mongoose/index.d.ts

```typescript
// ######################################## THIS FILE WAS GENERATED BY MONGOOSE-TSGEN ######################################## //

// NOTE: ANY CHANGES MADE WILL BE OVERWRITTEN ON SUBSEQUENT EXECUTIONS OF MONGOOSE-TSGEN.
// TO ADD CUSTOM INTERFACES, DEFINE THEM IN THE `CUSTOM INTERFACES` BLOCK

import mongoose from "mongoose";
type ObjectId = mongoose.Types.ObjectId;

declare module "mongoose" {

	interface IUserFriend extends mongoose.Types.Subdocument {
		uid: IUser["_id"] | IUser;
		nickname?: string;
	}

	export interface IUserModel extends Model<IUser> {
		getFriends: Function;
	}

	export interface IUser extends Document {
		email: string;
		metadata?: any;
		firstName: string;
		lastName: string;
		friends: Types.DocumentArray<IUserFriend>;
		cityCoordinates?: Types.Array<number>;
		name: any;
		isMetadataString: Function;
	}

// ########################################## CUSTOM INTERFACES ########################################## //
// ######################################## END CUSTOM INTERFACES ######################################## //
}
```

## Initializing Schemas

Once you've generated your index.d.ts file, all you need to do is add the following types to your schema definitions:

### user.ts before:

```typescript
import mongoose from "mongoose";

const UserSchema = new Schema(...);

export const User = mongoose.model("User", UserSchema);
export default User;
```

### user.ts after:

```typescript
import mongoose, { IUser, IUserModel } from "mongoose";

const UserSchema = new Schema(...);

export const User: IUserModel = mongoose.model<IUser, IUserModel>("User", UserSchema);
export default User;
```

Then you can import the interfaces across your application from the Mongoose module and use them for document types:

```typescript
// import interface from mongoose module
import { IUser } from "mongoose"

async function getUser(uid: string): IUser {
  // user will be of type IUser
  const user = await User.findById(uid);
  return user;
}

async function editEmail(user: IUser, newEmail: string): IUser {
  user.email = newEmail;
  return await user.save();
}
```