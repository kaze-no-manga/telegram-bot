# telegram-bot

> Telegram bot for Kaze no Manga - Track and read manga from Telegram

## Overview

This repository contains an experimental Telegram bot that allows users to interact with Kaze no Manga directly from Telegram, enabling manga search, library management, and reading progress tracking.

## Features

- 🔍 **Search**: Search manga by title
- 📚 **Library**: View and manage your manga library
- 📖 **Reading**: Get chapter updates and mark as read
- 🔔 **Notifications**: Receive new chapter alerts in Telegram
- 🔐 **Authentication**: Link Telegram account with Kaze no Manga
- ⚡ **Lambda-based**: Deployed as AWS Lambda function

## Commands

### `/start`

Initialize the bot and link your account.

```
/start
→ Welcome! Link your Kaze no Manga account to get started.
```

### `/search <query>`

Search for manga.

```
/search one piece
→ Found 5 results:
   1. One Piece (Ongoing, 1095 chapters)
   2. One Piece: Strong World
   ...
```

### `/library`

View your manga library.

```
/library
→ Your Library (12 manga):
   📖 One Piece - Chapter 1050/1095
   ✅ Naruto - Completed
   📅 Attack on Titan - Plan to Read
```

### `/add <manga_id>`

Add manga to your library.

```
/add 123
→ Added "One Piece" to your library!
```

### `/remove <manga_id>`

Remove manga from your library.

```
/remove 123
→ Removed "One Piece" from your library.
```

### `/chapters <manga_id>`

Get chapter list for a manga.

```
/chapters 123
→ One Piece - Latest Chapters:
   Chapter 1095: "The World's Strongest"
   Chapter 1094: "The Battle Continues"
   Chapter 1093: "Luffy vs Kaido"
```

### `/read <chapter_id>`

Mark chapter as read.

```
/read 456
→ Marked "One Piece - Chapter 1095" as read!
```

### `/progress <manga_id>`

Check your reading progress.

```
/progress 123
→ One Piece
   Current: Chapter 1050
   Latest: Chapter 1095
   Progress: 96%
```

### `/notifications`

Manage notification settings.

```
/notifications
→ Notification Settings:
   ✅ New chapters: ON
   ✅ Manga completed: ON
   Max per day: 10
   
   Use /notifications_off to disable
```

### `/help`

Show help message.

```
/help
→ Available commands:
   /search - Search manga
   /library - View your library
   /add - Add manga to library
   ...
```

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Telegram API                           │
│                  (Webhook → Lambda)                       │
└──────────────────────────────────────────────────────────┘
                          │
                          ▼
                    ┌──────────┐
                    │  Lambda  │
                    │   Bot    │
                    └──────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌─────────┐     ┌─────────┐     ┌─────────┐
    │ GraphQL │     │DynamoDB │     │   SES   │
    │   API   │     │(Sessions)│     │(Notify) │
    └─────────┘     └─────────┘     └─────────┘
```

## Implementation

### Bot Setup

```typescript
// src/index.ts
import { Telegraf } from 'telegraf'
import { APIGatewayProxyHandler } from 'aws-lambda'

const bot = new Telegraf(process.env.TELEGRAM_BOT_TOKEN!)

// Commands
bot.command('start', async (ctx) => {
  await ctx.reply('Welcome to Kaze no Manga! Use /help to see available commands.')
})

bot.command('search', async (ctx) => {
  const query = ctx.message.text.split(' ').slice(1).join(' ')
  if (!query) {
    return ctx.reply('Usage: /search <manga title>')
  }
  
  const results = await searchManga(query)
  const message = formatSearchResults(results)
  await ctx.reply(message)
})

bot.command('library', async (ctx) => {
  const userId = await getUserId(ctx.from.id)
  const library = await getLibrary(userId)
  const message = formatLibrary(library)
  await ctx.reply(message)
})

// Lambda handler
export const handler: APIGatewayProxyHandler = async (event) => {
  try {
    const update = JSON.parse(event.body!)
    await bot.handleUpdate(update)
    return { statusCode: 200, body: 'OK' }
  } catch (error) {
    console.error(error)
    return { statusCode: 500, body: 'Error' }
  }
}
```

### GraphQL Integration

```typescript
// src/services/graphql.ts
import { GraphQLClient } from 'graphql-request'

const client = new GraphQLClient(process.env.GRAPHQL_ENDPOINT!)

export async function searchManga(query: string) {
  const result = await client.request(`
    query SearchManga($query: String!) {
      searchManga(query: $query) {
        id
        title
        coverImage
        status
        latestChapter
      }
    }
  `, { query })
  
  return result.searchManga
}

export async function getLibrary(userId: string) {
  const result = await client.request(`
    query GetLibrary {
      getLibrary {
        id
        manga {
          id
          title
          coverImage
        }
        status
        currentChapterNumber
        lastReadAt
      }
    }
  `, {}, {
    authorization: `Bearer ${await getAuthToken(userId)}`
  })
  
  return result.getLibrary
}

export async function addToLibrary(userId: string, mangaId: string) {
  const result = await client.request(`
    mutation AddToLibrary($mangaId: ID!) {
      addToLibrary(mangaId: $mangaId) {
        id
        status
      }
    }
  `, { mangaId }, {
    authorization: `Bearer ${await getAuthToken(userId)}`
  })
  
  return result.addToLibrary
}
```

### Session Management

```typescript
// src/services/session.ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb'
import { DynamoDBDocumentClient, GetCommand, PutCommand } from '@aws-sdk/lib-dynamodb'

const client = DynamoDBDocumentClient.from(new DynamoDBClient({}))

export async function getUserId(telegramId: number): Promise<string | null> {
  const result = await client.send(new GetCommand({
    TableName: process.env.SESSIONS_TABLE!,
    Key: { telegramId }
  }))
  
  return result.Item?.userId || null
}

export async function linkAccount(telegramId: number, userId: string, authToken: string) {
  await client.send(new PutCommand({
    TableName: process.env.SESSIONS_TABLE!,
    Item: {
      telegramId,
      userId,
      authToken,
      createdAt: Date.now()
    }
  }))
}

export async function getAuthToken(userId: string): Promise<string> {
  // Retrieve auth token for user
}
```

### Message Formatting

```typescript
// src/utils/format.ts
export function formatSearchResults(results: Manga[]) {
  if (results.length === 0) {
    return 'No manga found.'
  }
  
  const lines = results.slice(0, 10).map((manga, i) => {
    const status = manga.status === 'ongoing' ? '📖' : '✅'
    return `${i + 1}. ${status} ${manga.title} (${manga.latestChapter} chapters)`
  })
  
  return `Found ${results.length} results:\n\n${lines.join('\n')}`
}

export function formatLibrary(library: LibraryEntry[]) {
  if (library.length === 0) {
    return 'Your library is empty. Use /search to find manga!'
  }
  
  const lines = library.map(entry => {
    const emoji = {
      reading: '📖',
      completed: '✅',
      plan_to_read: '📅',
      dropped: '❌',
      on_hold: '⏸️'
    }[entry.status]
    
    const progress = entry.currentChapterNumber 
      ? ` - Chapter ${entry.currentChapterNumber}`
      : ''
    
    return `${emoji} ${entry.manga.title}${progress}`
  })
  
  return `Your Library (${library.length} manga):\n\n${lines.join('\n')}`
}
```

### Notifications

```typescript
// src/services/notifications.ts
import { Telegraf } from 'telegraf'

const bot = new Telegraf(process.env.TELEGRAM_BOT_TOKEN!)

export async function notifyNewChapter(
  telegramId: number,
  manga: Manga,
  chapter: Chapter
) {
  const message = `
🆕 New Chapter!

${manga.title}
Chapter ${chapter.number}: ${chapter.title || 'Untitled'}

Use /read ${chapter.id} to mark as read.
  `.trim()
  
  await bot.telegram.sendMessage(telegramId, message)
}

export async function notifyMangaCompleted(
  telegramId: number,
  manga: Manga
) {
  const message = `
✅ Manga Completed!

${manga.title} has been marked as completed.

Find similar manga with /search
  `.trim()
  
  await bot.telegram.sendMessage(telegramId, message)
}
```

## Account Linking

### Flow

1. User sends `/start` to bot
2. Bot generates a unique linking code
3. User enters code on Kaze no Manga website
4. Website validates code and links accounts
5. Bot confirms successful linking

### Implementation

```typescript
// src/commands/start.ts
bot.command('start', async (ctx) => {
  const telegramId = ctx.from.id
  const existingUser = await getUserId(telegramId)
  
  if (existingUser) {
    return ctx.reply('Your account is already linked!')
  }
  
  // Generate linking code
  const code = generateLinkingCode()
  await saveLinkingCode(telegramId, code)
  
  const message = `
Welcome to Kaze no Manga!

To link your account:
1. Go to https://kazenomanga.com/settings/telegram
2. Enter this code: ${code}
3. Click "Link Account"

The code expires in 10 minutes.
  `.trim()
  
  await ctx.reply(message)
})
```

## Webhook Setup

```typescript
// Set webhook URL
const webhookUrl = 'https://api.kazenomanga.com/telegram/webhook'

await bot.telegram.setWebhook(webhookUrl)
```

## Environment Variables

```bash
# .env
TELEGRAM_BOT_TOKEN="123456:ABC-DEF..."
GRAPHQL_ENDPOINT="https://api.kazenomanga.com/graphql"
SESSIONS_TABLE="telegram-sessions"
```

## Development

```bash
# Install dependencies
npm install

# Build
npm run build

# Test locally (polling mode)
npm run dev

# Deploy (via CDK)
cd ../infrastructure
npm run deploy
```

## Testing

```bash
# Unit tests
npm test

# Test with local bot
npm run dev

# Test webhook
curl -X POST https://api.kazenomanga.com/telegram/webhook \
  -H "Content-Type: application/json" \
  -d '{"message": {"text": "/start"}}'
```

## Deployment

Deployed as AWS Lambda function via CDK:

```typescript
// In infrastructure repo
const telegramLambda = new lambda.Function(this, 'TelegramBot', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('path/to/telegram-bot/dist'),
  environment: {
    TELEGRAM_BOT_TOKEN: telegramBotToken.secretValue.toString(),
    GRAPHQL_ENDPOINT: api.graphqlUrl,
    SESSIONS_TABLE: sessionsTable.tableName,
  },
})

// API Gateway for webhook
const api = new apigateway.RestApi(this, 'TelegramWebhookApi')
const webhook = api.root.addResource('webhook')
webhook.addMethod('POST', new apigateway.LambdaIntegration(telegramLambda))
```

## Use Cases

- **Quick Search**: Search manga without opening the app
- **Library Management**: Add/remove manga on the go
- **Progress Tracking**: Mark chapters as read from Telegram
- **Notifications**: Get instant alerts for new chapters
- **Reading Reminders**: Bot can remind you to continue reading

## Limitations

- **Experimental**: This is an experimental feature
- **No Image Display**: Cannot display manga pages in Telegram
- **Rate Limits**: Subject to Telegram API rate limits
- **Text-Only**: All interactions are text-based

## Future Enhancements

- [ ] Inline search (type `@kazenomanga one piece`)
- [ ] Keyboard buttons for common actions
- [ ] Reading statistics and insights
- [ ] Group chat support (shared libraries)
- [ ] Voice commands
- [ ] Image previews (cover images)

## License

MIT License - see [LICENSE](LICENSE) for details.

---

**Part of the [Kaze no Manga](https://github.com/kaze-no-manga) project**
