-- Better Lobby Management: English texts, the source of every translation.
-- Translators: see TRANSLATING.md. The comment above an entry says where the
-- text shows. Keys under option. appear in Mod Options Menu, which upper-cases
-- the mod name and choices. ON and OFF are the game's own words and are not here.
return {
    mod = 'better_lobby_management',
    title = 'Better Lobby Management',
    language = 'zh-Hant',
    strings = {
        -- Escape-menu button (GAME tab, host of a squad on the ship). Upper case like the
        -- game's own buttons. Long labels scroll inside the button.
        ['button.disband'] = '解散小隊',
        -- Escape-menu button. {name} is the chosen player's name, upper-cased.
        ['button.promote'] = '將隊長轉讓給{name}',
        -- Escape-menu button (host in a mission while their SOS Beacon is on).
        ['button.cancel_sos'] = '取消SoS信標',

        -- Title of the game's confirm dialog for DISBAND SQUAD.
        ['dialog.disband.title'] = '解散小隊',
        -- Its text. The dialog holds about three lines.
        ['dialog.disband.body'] = '踢出其他玩家，他們會回到自己的船上。',
        -- Title of the confirm dialog for PROMOTE. {name}: the player, upper-cased.
        ['dialog.promote.title'] = '將隊長轉讓給{name}',
        -- Its text when the host picked the player (by opening their player menu).
        -- {name}: the player's name as written.
        ['dialog.promote.body'] = '{name}將在其船上擔任隊長，你和其他人會一同前往。',
        -- Its text when the mod picked the player.
        ['dialog.promote.body_automatic'] = '{name}將在其船上擔任隊長，你和其他人會一同前往。此次人選爲自動挑選，可以選擇玩家卡片以手動挑選。',
        -- Title of the confirm dialog for CANCEL SOS.
        ['dialog.cancel_sos.title'] = '取消SoS信標',
        -- Its text for a Public lobby.
        ['dialog.cancel_sos.public'] = '就算小隊還有空位，也依然取消SoS信標。若配對隱私設置爲公開，快速匹配依然能搜索到你的任務。可再次呼叫SoS信標。',
        -- Its text for other lobbies. {privacy} is one of the four texts below.
        ['dialog.cancel_sos.private'] = '就算小隊還有空位，也依然取消SoS信標。配對隱私變更回{privacy}。可再次呼叫SoS信標。',
        -- The game's privacy settings, as the game names them.
        ['privacy.friends'] = '僅限好友',
        ['privacy.invite'] = '僅限受邀者',
        ['privacy.clan'] = 'Friends and Clan',
        -- When the setting is not one of the three above.
        ['privacy.other'] = '其原本設置',
        -- A player whose name is unknown. {id} is a number in hexadecimal.
        ['player.unknown'] = '玩家{id}',

        -- Chat line sent to the whole squad, as from the host, before DISBAND SQUAD
        -- kicks them. It goes out in the host's language.
        ['chat.disband'] = 'The host disbanded the squad.',

        -- Mod Options Menu: the mod's name (upper-cased by the menu).
        ['option.mod'] = '優化小隊管理',
        ['option.region.label'] = '匹配地區',
        ['option.region.default'] = '默認',
        ['option.region.continent'] = '僅限同一大洲',
        ['option.region.description'] = '使用掃描器和快速匹配時，僅搜索和你處於同一大洲的玩家（遊戲判定爲低延遲的玩家）創建的任務。不會增加搜索次數，且可能導致搜索到的任務變少。',
        ['option.messages.label'] = '發送通知',
        ['option.messages.description'] = '踢出玩家前，會發送通知告知原因。轉讓隊長會發送遊戲原有的轉讓隊長通知，解散小隊會由你發出聊天消息。',
        ['option.scanner.label'] = '掃描器充電時長（秒）',
        ['option.scanner.description'] = '銀河地圖中，尋找任務的掃描器充電的所需時長，可設置爲5至20，遊戲默認值爲20。本模組不會讓時長超過遊戲默認值。每次掃描即爲一次搜索，所以值越小，搜索越頻繁。',
    },
    -- Mod Options Menu's limits, in characters.
    limits = {
        ['option.mod'] = 40, ['option.region.label'] = 64, ['option.messages.label'] = 64,
        ['option.scanner.label'] = 64, ['option.region.default'] = 48, ['option.region.continent'] = 48,
        ['option.region.description'] = 400, ['option.messages.description'] = 400,
        ['option.scanner.description'] = 400,
    },
    -- Rough room in ems for dialog titles (the tool warns when a translation looks
    -- wider). Button labels have none: they scroll.
    widths = {['dialog.disband.title'] = 22, ['dialog.cancel_sos.title'] = 22},
}
